### 基于PI4J在树莓派上为一些硬件写的驱动程序

> 在开发污水处理设备远程监控项目时， 使用树莓派的Serial端口，通过RS485对一些硬件进行通讯, 并将数据上传到阿里云。 对一些电流互感器、智能电表、电动机保护器等硬件通过 MODBUS RTU 协议进行访问控制的一些过时代码或不用的硬件，其相关驱动程序存放在此留作备份

#### 10路 交流电流 监测器
``` java
package com.ryme.device.serial;

import com.aliyun.alink.linksdk.tmp.device.payload.ValueWrapper;
import com.pi4j.io.serial.SerialDataEventListener;
import com.ryme.Bootstrap;
import com.ryme.device.DeviceInterface;
import com.ryme.dictionary.DeviceName;
import com.ryme.dictionary.DeviceType;
import com.ryme.error.SystemRuntimeException;
import lombok.extern.slf4j.Slf4j;

import java.io.IOException;
import java.util.Arrays;
import java.util.Map;
import java.util.concurrent.TimeUnit;

import static com.ryme.util.ModbusUtil.appendCRC16;
import static com.ryme.util.ModbusUtil.byteToInt16;
import static com.ryme.util.ModbusUtil.checkCRC16;

/**
 * Description : 10路 交流电流 监测器
 * Created by Setsuna Jin on 2019/1/25.
 */
@Slf4j
@Deprecated
public class CurrentMonitor implements DeviceInterface {
    // 设备地址
    private byte address;
    // 设备名称
    private DeviceName deviceName;
    // 设备属性
    private double[] currents;

    public interface CurrentProcess{
        void process(double[] currents);
    }

    public CurrentMonitor(DeviceName deviceName, Byte address, CurrentProcess currentProcess){
        // 初始化配置项
        this.deviceName = deviceName;
        this.address = address;
        this.currents = new double[10];
        // 登记设备信息
        DEVICES.put(deviceName.name(), this);
        // 添加监听器
        final SerialDataEventListener serialDataEventListener = event -> {
            try {
                byte[] bytes = event.getBytes();
                if (bytes[0] != address)
                    return;
                log.info("设备:{} 处理电流数据响应 开始", deviceName);
                // 校验 CRC16
                if (!checkCRC16(bytes)){
                    throw new SystemRuntimeException("设备:{} 响应结果 CRC16校验失败:{}", deviceName, event.getHexByteString());
                }
                // 读取寄存器 得到的结果
                if (bytes[1] == 0x03){
                    // 取出 数据段字节
                    byte[] payload = Arrays.copyOfRange(bytes, 3, bytes.length - 2);
                    // 正常接收的数据字节数应为20个字节
                    if (bytes[2] == 20){
                        for (int i = 0; i < 10; i++) {
                            // 转换字节为电流, 两位小数
                            this.currents[i] = byteToInt16(payload[i*2], payload[i*2+1]) * 1.0 / 100;
                            // 分辨率 为 0.1A, 当小于该值时按 0A 计算
                            if (this.currents[i] < 0.1d){
                                this.currents[i] = 0.0d;
                            }
                        }
                        // 处理电流信息
                        currentProcess.process(this.currents);
                    }
                    else
                        throw new SystemRuntimeException("无法解析 设备:{} 的响应结果:{}", deviceName, event.getHexByteString());
                }
            } catch (Exception e) {
                log.error("设备:{} 处理电流数据响应 异常", deviceName);
                e.printStackTrace();
            }
            log.info("设备:{} 处理电流数据响应 结束", deviceName);
        };
        Bootstrap.SERIAL.addListener(serialDataEventListener);
        // 添加定时任务
        final Runnable autoRequest = ()->{
            log.info("设备:{} 读取电流寄存器内容请求 开始", deviceName);
            try {
                // 请求 读取 0x0000 开始的 10 个寄存器
                Bootstrap.SERIAL.write(appendCRC16(new byte[]{address, 0x03, 0x00, 0x00, 0x00, 0x0A}));
                Bootstrap.SERIAL.flush();
            } catch (IOException e) {
                log.error("设备:{} 读取电流寄存器内容请求 异常", deviceName);
                e.printStackTrace();
            }
            log.info("设备:{} 读取电流寄存器内容请求 结束", deviceName);
        };
        Bootstrap.TASK.scheduleAtFixedRate(autoRequest,500, 1000, TimeUnit.MILLISECONDS);
    }

    /**
     * 设备的初始化参数
     *
     * @return Map<String, ValueWrapper>
     */
    @Override
    public Map<String, ValueWrapper> initProperty() {
        return null;
    }

    /**
     * 设备当前的参数
     *
     * @return Map<String, ValueWrapper>
     */
    @Override
    public Map<String, ValueWrapper> getProperty() {
        return null;
    }

    /**
     * 设置设备的参数
     *
     * @param key   参数名称
     * @param value 参数内容
     */
    @Override
    public void setProperty(String key, Object value) {
        // 该设备不支持属性设置
        throw new SystemRuntimeException("设备:{} 不支持设置参数 {}:{}", deviceName, key, value);
    }

    /**
     * 获取设备名称
     *
     * @return 设备名称
     */
    @Override
    public DeviceName getName() {
        return this.deviceName;
    }

    /**
     * 获取设备类型
     *
     * @return 设备类型
     */
    @Override
    public DeviceType getType() {
        return DeviceType.SERIAL_CURRENT_MONITOR;
    }
}
```

#### 电动机智能保护器 JN300Z10TL
``` java
package com.ryme.device.serial;

import com.aliyun.alink.linksdk.tmp.device.payload.ValueWrapper;
import com.pi4j.io.serial.SerialDataEventListener;
import com.ryme.Bootstrap;
import com.ryme.alink.LinkKitEventPost;
import com.ryme.device.DeviceInterface;
import com.ryme.device.gpio.Relay;
import com.ryme.dictionary.DeviceName;
import com.ryme.dictionary.DeviceType;
import com.ryme.error.SystemRuntimeException;
import com.ryme.util.StringUtil;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.extern.slf4j.Slf4j;

import java.io.IOException;
import java.util.*;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

import static com.ryme.Bootstrap.TASK;
import static com.ryme.util.ModbusUtil.appendCRC16;
import static com.ryme.util.ModbusUtil.byteToInt16;
import static com.ryme.util.ModbusUtil.checkCRC16;

/**
 * Description : 单个电机保护器
 * Created by Setsuna Jin on 2019/5/25.
 */
@Slf4j
@Deprecated
public class JN300Z10TL implements DeviceInterface {

    private static final String[] ErrorNames = new String[]{
            "未启动完成", "未跳闸故障", "故障", "电机启动", "漏电", "短路", "欠压 或 过压", "欠流", "断相", "不平衡", "堵转", "过流",
    };

    // 数据名称
    @Getter
    @AllArgsConstructor
    public enum DATA_NAME {
        // SECTION_A ---------------------------------------------------------------------------------------------------
        // 0x0001 前八位
        NOT_FINISH_START(1),               // 未启动完成
        NOT_TRIP_ERROR(1),                 // 未跳闸故障
        ERROR(1),                          // 故障
        STARTED(1),                        // 电机启动
        // 0x0001 后八位
        ELECTRIC_LEAKAGE(1),               // 漏电
        SHORT_CIRCUIT(1),                  // 短路
        UNDER_OR_OVER_VOLTAGE(1),          // 欠压 或 过压
        UNDER_CURRENT(1),                  // 欠流
        PHASE_LOSS(1),                     // 断相
        IMBALANCE(1),                      // 不平衡
        LOCKED_ROTOR(1),                   // 堵转
        OVER_CURRENT(1),                   // 过流
        // 0x0002 - 0x0007
        PHASE_A_CURRENT(100),              // A 相电流
        PHASE_B_CURRENT(100),              // B 相电流
        PHASE_C_CURRENT(100),              // C 相电流
        LEAKAGE_CURRENT(1),                // 漏电电流值
        VOLTAGE(1),                        // 电压值
        SPECIFICATION(1),                  // 规格
        ;
        private int decimal;
    }

    // 查询的数据片段
    // 7 个寄存器 14 字节
    private static final byte SECTION_A = 0x0E;

    // 设备地址
    private byte address;
    // 设备名称
    private DeviceName deviceName;
    // 设备属性
    private Map<String, ValueWrapper> properties;

    // 事件回调
    private Runnable onEnable = null;
    private Runnable onDisable = null;
    private Runnable onError = null;

    // 异常缓冲计数器
    private int errorBuffer = 2;
    // 异常时间记录器
    private List<Long> errorTimes;

    // 启动时间计数器
    private Long enabledAt = null;

    // 启动时间计数器
    private Long disableAt = null;

    /**
     * 错误信息转换
     * @param c1 控制字1
     * @param c2 控制字2
     * @return 错误信息
     */
    public String errorMessageParse(String c1, String c2){
        String controlBinary = (c1 + c2).substring(4);
        StringBuilder stringBuilder = new StringBuilder();
        for (int i = 0; i < ErrorNames.length; i++) {
            if (controlBinary.charAt(i) == '1') {
                stringBuilder.append(ErrorNames[i]).append(", ");
            }
        }
        String errorMessage = stringBuilder.toString().replaceFirst(", $", "");
        Relay ass = searchAss();
        if (Objects.nonNull(ass)) {
            errorMessage = String.format("%s部件 %s", ass.getStatus().isHigh()? "备用" : "主用", errorMessage);
        }
        return errorMessage;
    }

    /**
     * 主备切换继电器 查询
     * TODO: 不符合设计原则, 待优化设计
     * @return 继电器
     */
    public Relay searchAss() {
        DeviceInterface ass = DEVICES.get(deviceName.name().replace("PROTECTOR", "ASS"));
        if (Objects.nonNull(ass))
            return (Relay) ass;
        else
            return null;
    }

    private int startCheck(long currentTime, boolean isStarted) {
        // 如 30 秒内 设备应启动但未启动起来 视为异常
        if (Objects.nonNull(enabledAt) && !isStarted && (currentTime - enabledAt) / 1000 > 30) {
            return 1;
        }
        // 如 30 秒内 设备应停止但未仍在运行 视为异常
        if (Objects.nonNull(disableAt) && isStarted && (currentTime - disableAt) / 1000 > 30) {
            return 2;
        }
        return 0;
    }

    private void errorReport(LinkKitEventPost.EVENT_TYPE type, boolean errorCheck, int startCheck, String c1, String c2){
        Relay ass = searchAss();
        if (startCheck == 1) {
            LinkKitEventPost.error(type, "超过30秒仍无法启动", this, ass);
        } else if (startCheck == 2) {
            LinkKitEventPost.error(type, "超过30秒仍无法停止", this, ass);
        }
        if (errorCheck)
            LinkKitEventPost.error(type, this.errorMessageParse(c1, c2), this, ass);
    }

    private void errorCheck(String c1, String c2) {
        long currentTime = System.currentTimeMillis();
        // 异常类型
        int startCheck = startCheck(currentTime, c1.charAt(7) == '1');
        boolean errorCheck = c1.charAt(6) == '1';
        // 异常计数器
        if (errorCheck || startCheck > 0) {
            if (errorBuffer > 0) {
                log.error("设备[{}] 出现待确认故障, 故障代码: {} {}", deviceName, c1, c2);
                errorBuffer -= 1;
                // 事件上报
                this.errorReport(LinkKitEventPost.EVENT_TYPE.ERROR_INFO, errorCheck, startCheck, c1, c2);
            } else {
                log.error("设备[{}] 出现已确认故障, 故障代码: {} {}", deviceName, c1, c2);
                // 延时1s后关闭设备
                TASK.schedule(() -> {
                    this.disable();
                    if (Objects.nonNull(onError))
                        onError.run();
                }, 1, TimeUnit.SECONDS);
                errorTimes.add(currentTime);
                // 事件上报
                this.errorReport(LinkKitEventPost.EVENT_TYPE.ERROR_WARN, errorCheck, startCheck, c1, c2);
            }
        } else {
            errorBuffer = 2;
        }

        // 如果在一分钟内出现了 3 次以上错误 则触发严重异常进而宕机
        long oneMinutesAgo = currentTime - 60 * 1000;
        List<Long> collect = errorTimes.stream().filter(t -> t > oneMinutesAgo).collect(Collectors.toList());
        // 防止 errorTimes 过多
        if (errorTimes.size() > 30) {
            errorTimes.removeAll(errorTimes.stream().filter(t -> t < oneMinutesAgo).collect(Collectors.toList()));
        }
        if (collect.size() > 2) {
            // 事件上报
            this.errorReport(LinkKitEventPost.EVENT_TYPE.ERROR_DOWN, errorCheck, startCheck, c1, c2);
            // 逻辑宕机
            Bootstrap.LOGIC.breakDown(String.format("%s 出现严重异常", deviceName.name()));
        }
    }


    /**
     * 控制设备启停
     * @param isStart 是否启动
     */
    private void setStart(Boolean isStart) {
        String status = isStart ? "启动" : "停止";
        byte[] start = appendCRC16( isStart ?
                // 启动指令
                new byte[]{address, 0x05, 0x00, 0x01, (byte) 0xFF, 0x00} :
                // 停止指令
                new byte[]{address, 0x05, 0x00, 0x00, 0x00, 0x00});
        log.info("设备[{}] {} 指令:{}", deviceName, status, StringUtil.bytesToHex(start, true));
        try {
            Bootstrap.SERIAL.write(start);
            Bootstrap.SERIAL.flush();
        } catch (IOException e) {
            log.error("设备[" + deviceName + "] "+ status + "失败", e);
        }
    }

    public void enable(){
        this.setStart(true);
        enabledAt = System.currentTimeMillis();
        disableAt = null;
        if (Objects.nonNull(onEnable))
            onEnable.run();
    }

    public void disable() {
        this.setStart(false);
        enabledAt = null;
        disableAt = System.currentTimeMillis();
        if (Objects.nonNull(onDisable))
            onDisable.run();
    }

    /**
     * 初始化通讯
     */
    private void initConnection(){
        try {
            byte[] init = appendCRC16(new byte[]{address, 0x03, 0x00, 0x00, 0x00, 0x01});
            log.info("设备[{}] 通讯初始化 指令:{}", deviceName, StringUtil.bytesToHex(init, true));
            Bootstrap.SERIAL.write(init);
            Bootstrap.SERIAL.flush();
        } catch (IOException e) {
            log.error("通讯初始化失败", e);
        }
    }

    public JN300Z10TL(int initialDelay, byte address, DeviceName deviceName) {
        this(initialDelay, address, deviceName, null, null, null);
    }

    public JN300Z10TL(int initialDelay, byte address, DeviceName deviceName, Runnable onError) {
        this(initialDelay, address, deviceName, null, null, onError);
    }

    public JN300Z10TL(int initialDelay, byte address, DeviceName deviceName, Runnable onEnable, Runnable onDisable, Runnable onError){
        // 初始化配置项
        this.deviceName = deviceName;
        this.address = address;
        this.properties = getInitProperty();
        this.onEnable = onEnable;
        this.onDisable = onDisable;
        this.onError = onError;
        this.errorTimes = new ArrayList<>();
        // 登记设备信息
        DEVICES.put(deviceName.name(), this);
        // 构建监听器
        final SerialDataEventListener serialDataEventListener = event -> {
            try {
                byte[] bytes = event.getBytes();
                if (bytes.length == 0 || bytes[0] != this.address)
                    return;
                if (log.isDebugEnabled()) {
                    log.debug("设备:{} 接收到响应: {}", deviceName, event.getHexByteString());
                    log.debug("设备:{} 处理数据响应 开始", deviceName);
                }
                // 校验 CRC16
                if (!checkCRC16(bytes)){
                    throw new SystemRuntimeException("设备:{} 响应结果 CRC16校验失败:{}", deviceName, event.getHexByteString());
                }
                // 读取寄存器 得到的结果
                if (bytes[1] == 0x03){
                    // 取出 数据段字节
                    byte[] payload = Arrays.copyOfRange(bytes, 3, bytes.length - 2);
                    if (bytes[2] == SECTION_A && payload.length == SECTION_A)
                        this.processSectionA(payload);
                }
            } catch (Exception e) {
                log.error("设备:"+ deviceName +" 处理数据响应 异常", e);
                // 如果出现读取异常则将数据重置为初始值
                this.properties = getInitProperty();
            }
            log.debug("设备:{} 处理数据响应 结束", deviceName);
        };
        // 添加监听器
        Bootstrap.SERIAL.addListener(serialDataEventListener);
        // 通讯初始化
        this.initConnection();
        // 添加定时任务
        Bootstrap.TASK.scheduleAtFixedRate(this.generateSchedule((byte) 0x01, (byte) 0x07), initialDelay, 5, TimeUnit.SECONDS);
    }

    /**
     * 生成定时任务
     * @param start 开始读取的寄存器位置
     * @param count 读取寄存器的数量
     * @return 定时任务
     */
    private Runnable generateSchedule(byte start, byte count) {
        return ()->{
//            log.debug("设备:{} 读取寄存器内容 请求 开始", deviceName);
//            try {
//                // 每次最多只能读10个寄存器
//                byte[] command = appendCRC16(new byte[]{address, 0x03, 0x00, start, 0x00, count});
//                if (log.isDebugEnabled()) {
//                    log.debug("设备[{}] 读取 0x00{} 开始的  {} 个寄存器 指令:{}", deviceName,
//                            Integer.toHexString(start), Integer.toString(count), StringUtil.bytesToHex(command, true));
//                }
//                Bootstrap.SERIAL.write(command);
//                Bootstrap.SERIAL.flush();
//            } catch (IOException e) {
//                log.error("设备:[" + deviceName + "] 读取寄存器内容 请求 异常", e);
//            }
//            log.debug("设备:[{}] 读取寄存器内容 请求 结束", deviceName);
        };
    }

    private void processSectionA(byte[] payload) {
        if (log.isDebugEnabled())
            log.debug("Section A: {}", StringUtil.bytesToHex(payload, true));
        // 0 1 控制字 二进制字符串 转 布尔值 12 个布尔值
        String c1 = StringUtil.byteToBinaryString(payload[0]);
        String c2 = StringUtil.byteToBinaryString(payload[1]);

        // 标记为 1 的为状态有效值
        log.debug("控制字1: {}", c1); // 0000 1111
        for (int i = 0; i < 4; i++) {
            char status = c1.charAt(4 + i);
            properties.put(getPropName(DATA_NAME.values()[i]), new ValueWrapper.BooleanValueWrapper(Integer.parseInt(status + "")));
        }
        log.debug("控制字2: {}", c2); // 1111 1111
        for (int i = 4; i < 12; i++) {
            char status = c1.charAt(i - 4);
            properties.put(getPropName(DATA_NAME.values()[i]), new ValueWrapper.BooleanValueWrapper(Integer.parseInt(status + "")));
        }
        // 2 3 | 4 5 | 6 7 | 8 9 | 10 11
        for (int i = 0; i < 5; i++) {
            DATA_NAME dataName = DATA_NAME.values()[i + 12];
            // 16 进制按十进制数使用
            byte[] bytes = new byte[]{payload[i*2 + 2], payload[i*2 + 3]};
            double value = Integer.parseInt(StringUtil.bytesToHex(bytes, false)) * 1.0 / dataName.getDecimal();
            properties.put(getPropName(dataName), new ValueWrapper.DoubleValueWrapper(value));
        }
        // 12 13
        properties.put(getPropName(DATA_NAME.SPECIFICATION), new ValueWrapper.IntValueWrapper(byteToInt16(payload[12], payload[13])));
        // 异常检查
        this.errorCheck(c1, c2);
    }

    private Map<String, ValueWrapper> getInitProperty() {
        properties = new HashMap<>();
        properties.put(getPropName(DATA_NAME.NOT_FINISH_START), new ValueWrapper.BooleanValueWrapper(0));
        properties.put(getPropName(DATA_NAME.NOT_TRIP_ERROR), new ValueWrapper.BooleanValueWrapper(0));
        properties.put(getPropName(DATA_NAME.ERROR), new ValueWrapper.BooleanValueWrapper(0));
        properties.put(getPropName(DATA_NAME.STARTED), new ValueWrapper.BooleanValueWrapper(0));

        properties.put(getPropName(DATA_NAME.ELECTRIC_LEAKAGE), new ValueWrapper.BooleanValueWrapper(0));
        properties.put(getPropName(DATA_NAME.SHORT_CIRCUIT), new ValueWrapper.BooleanValueWrapper(0));
        properties.put(getPropName(DATA_NAME.UNDER_OR_OVER_VOLTAGE), new ValueWrapper.BooleanValueWrapper(0));
        properties.put(getPropName(DATA_NAME.UNDER_CURRENT), new ValueWrapper.BooleanValueWrapper(0));
        properties.put(getPropName(DATA_NAME.PHASE_LOSS), new ValueWrapper.BooleanValueWrapper(0));
        properties.put(getPropName(DATA_NAME.IMBALANCE), new ValueWrapper.BooleanValueWrapper(0));
        properties.put(getPropName(DATA_NAME.LOCKED_ROTOR), new ValueWrapper.BooleanValueWrapper(0));
        properties.put(getPropName(DATA_NAME.OVER_CURRENT), new ValueWrapper.BooleanValueWrapper(0));

        properties.put(getPropName(DATA_NAME.PHASE_A_CURRENT), new ValueWrapper.DoubleValueWrapper(0d));
        properties.put(getPropName(DATA_NAME.PHASE_B_CURRENT), new ValueWrapper.DoubleValueWrapper(0d));
        properties.put(getPropName(DATA_NAME.PHASE_C_CURRENT), new ValueWrapper.DoubleValueWrapper(0d));
        properties.put(getPropName(DATA_NAME.LEAKAGE_CURRENT), new ValueWrapper.DoubleValueWrapper(0d));
        properties.put(getPropName(DATA_NAME.VOLTAGE), new ValueWrapper.DoubleValueWrapper(0d));
        properties.put(getPropName(DATA_NAME.SPECIFICATION), new ValueWrapper.IntValueWrapper(0));
        return properties;
    }

    /**
     * 设备的初始化参数
     * @return Map<String, ValueWrapper>
     */
    @Override
    public Map<String, ValueWrapper> initProperty() {
        return this.getInitProperty();
    }

    /**
     * 设备当前的参数
     * @return Map<String, ValueWrapper>
     */
    @Override
    public Map<String, ValueWrapper> getProperty() {
        return this.properties;
    }

    /**
     * 设置设备的参数
     * @param key   参数名称
     * @param value 参数内容
     */
    @Override
    public void setProperty(String key, Object value) {
        if (StringUtil.isNotBlank(key) && Objects.nonNull(value)){
            // 如果是设置 继电器是否启用
            if (getPropName(DATA_NAME.STARTED).equals(key)){
                if ((Integer) value == 1)
                    enable();
                else
                    disable();
            } else {
                // 该设备不支持属性设置
                throw new SystemRuntimeException("设备:{} 不支持设置参数 {}:{}", deviceName, key, value);
            }
        }
    }

    /**
     * 获取设备名称
     * @return 设备名称
     */
    @Override
    public DeviceName getName() {
        return this.deviceName;
    }

    /**
     * 获取设备类型
     * @return 设备类型
     */
    @Override
    public DeviceType getType() {
        return DeviceType.SERIAL_PUMP_MONITOR_JN300Z10TL;
    }
}
```

#### 智能电表 PDM803DL
``` java
package com.ryme.device.serial;

import com.aliyun.alink.linksdk.tmp.device.payload.ValueWrapper;
import com.pi4j.io.serial.SerialDataEventListener;
import com.ryme.Bootstrap;
import com.ryme.alink.LinkKitEventPost;
import com.ryme.device.DeviceInterface;
import com.ryme.dictionary.DeviceName;
import com.ryme.dictionary.DeviceType;
import com.ryme.error.SystemRuntimeException;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.extern.slf4j.Slf4j;

import java.io.IOException;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.TimeUnit;

import static com.ryme.util.ModbusUtil.appendCRC16;
import static com.ryme.util.ModbusUtil.byteToInt16;
import static com.ryme.util.ModbusUtil.checkCRC16;

/**
 * Description : PDM 803DL 电表设备
 * Created by Setsuna Jin on 2019/1/12.
 */
@Slf4j
@Deprecated
public class PDM803DL implements DeviceInterface {

    // 数据名称
    @Getter @AllArgsConstructor
    public enum DATA_NAME {
        // SECTION_A ---------------------------------------------------------------------------------------------------
        /**2*/ IA(1000), IB(1000), IC(1000), /** 00 00 */ UAN(10), UBN(10), UCN(10),
        // SECTION_B ---------------------------------------------------------------------------------------------------
        /**2*/ W(1000), VAR(1000), VA(1000), PF(1000), FREQ(100), WA(1000), WB(1000), WC(1000), VARA(1000), VARB(1000),
        VARC(1000), VAA(1000), VAB(1000), VAC(1000), PFA(1000), PFB(1000), PFC(1000), /**4*/ KWH(10),
        // SECTION_C ---------------------------------------------------------------------------------------------------
        /**4*/ KVARH(10), KWH_J(10), KVARH_J(10), KWH_F(10), KVARH_F(10), KWH_P(10), KVARH_P(10),
        KWH_G(10), KVARH_G(10), _KWH_HEX(10), _KVARH_HEX(10);
        // -------------------------------------------------------------------------------------------------------------
        private int decimal;
    }

    // 查询的数据片段
    // 14 字节
    private static final byte SECTION_A = 0x0E;
    // 40 字节
    private static final byte SECTION_B = 0x26;
    // 44 字节
    private static final byte SECTION_C = 0x2C;

    // 设备地址
    private byte address;
    // 设备名称
    private DeviceName deviceName;
    // 设备属性
    private Map<String, ValueWrapper> properties;

    // 最高电压
    private Double maxVoltage;
    // 最低电压
    private Double minVoltage;
    // 最大电流
    private Double maxCurrent;


    /**
     * 电流电压 检查预警
     */
    public boolean isVoltageAndCurrentOk(){
        boolean isOk = true;
        Double iA = (Double)properties.get(getPropName(DATA_NAME.IA)).getValue();
        Double iB = (Double)properties.get(getPropName(DATA_NAME.IB)).getValue();
        Double iC = (Double)properties.get(getPropName(DATA_NAME.IC)).getValue();
        Double vA = (Double)properties.get(getPropName(DATA_NAME.UAN)).getValue();
        Double vB = (Double)properties.get(getPropName(DATA_NAME.UBN)).getValue();
        Double vC = (Double)properties.get(getPropName(DATA_NAME.UCN)).getValue();
        // 检查最大电压
        if (Objects.nonNull(maxVoltage) && maxVoltage > 0){
            if (vA > maxVoltage || vB > maxVoltage || vC > maxVoltage){
                isOk = false;
                LinkKitEventPost.warnError("最大电压检查失败", this);
            }
        }
        // 检查最小电压
        if (Objects.nonNull(minVoltage) && minVoltage > 0){
            if (vA < minVoltage || vB < minVoltage || vC < minVoltage){
                isOk = false;
                LinkKitEventPost.warnError("最小电压检查失败", this);
            }
        }
        // 检查最大电流
        if (Objects.nonNull(maxCurrent) && maxCurrent > 0){
            if (iA > maxCurrent || iB > maxCurrent || iC > maxCurrent){
                isOk = false;
                LinkKitEventPost.warnError("最大电流检查失败", this);
            }
        }
        return isOk;
    }

    public PDM803DL(DeviceName deviceName, Byte address, Double maxVoltage, Double minVoltage, Double maxCurrent){
        // 初始化配置项
        this.deviceName = deviceName;
        this.address = address;
        this.properties = getInitProperty();
        // 初始化预警值
        this.maxVoltage = maxVoltage;
        this.minVoltage = minVoltage;
        this.maxCurrent = maxCurrent;
        // 登记设备信息
        DEVICES.put(deviceName.name(), this);
        // 添加监听器
        final SerialDataEventListener serialDataEventListener = event -> {
            try {
                byte[] bytes = event.getBytes();
                if (bytes[0] != address)
                    return;
                log.info("设备:{} 处理电表数据响应 开始", deviceName);
                // 校验 CRC16
                if (!checkCRC16(bytes)){
                    throw new SystemRuntimeException("设备:{} 响应结果 CRC16校验失败:{}", deviceName, event.getHexByteString());
                }
                // 读取寄存器 得到的结果
                if (bytes[1] == 0x03){
                    // 取出 数据段字节
                    byte[] payload = Arrays.copyOfRange(bytes, 3, bytes.length - 2);
                    if (bytes[2] == SECTION_A && payload.length == SECTION_A)
                        this.processSectionA(payload);
                    else if (bytes[2] == SECTION_B && payload.length == SECTION_B)
                        this.processSectionB(payload);
                    else if (bytes[2] == SECTION_C && payload.length == SECTION_C)
                        this.processSectionC(payload);
                    else
                        throw new SystemRuntimeException("无法解析 设备:{} 的响应结果:{}", deviceName, event.getHexByteString());
                }
                // 检查电流电压是否符合预警条件
                this.isVoltageAndCurrentOk();
            } catch (Exception e) {
                log.error("设备:{} 处理电表数据响应 异常", deviceName);
                e.printStackTrace();
                // 如果出现读取异常则将数据重置为初始值
                this.properties = getInitProperty();
            }
            log.info("设备:{} 处理电表数据响应 结束", deviceName);
        };
        Bootstrap.SERIAL.addListener(serialDataEventListener);
        // 添加定时任务
        final Runnable autoRequest = ()->{
            log.info("设备:{} 读取电表寄存器内容请求 开始", deviceName);
            try {
                // 请求 读取 0x0010 开始的  7 个寄存器 SECTION_A
                Bootstrap.SERIAL.write(appendCRC16(new byte[]{address, 0x03, 0x00, 0x10, 0x00, 0x07}));
                Bootstrap.SERIAL.flush();
                Thread.sleep(100);
                // 请求 读取 0x001A 开始的 19 个寄存器 SECTION_B
                Bootstrap.SERIAL.write(appendCRC16(new byte[]{address, 0x03, 0x00, 0x1A, 0x00, 0x13}));
                Bootstrap.SERIAL.flush();
                Thread.sleep(100);
                // 请求 读取 0x0032 开始的 22 个寄存器 SECTION_C
                Bootstrap.SERIAL.write(appendCRC16(new byte[]{address, 0x03, 0x00, 0x32, 0x00, 0x16}));
                Bootstrap.SERIAL.flush();
                Thread.sleep(100);
            } catch (IOException | InterruptedException e) {
                log.error("设备:{} 读取电表寄存器内容请求 异常", deviceName);
                e.printStackTrace();
            }
            log.info("设备:{} 读取电表寄存器内容请求 结束", deviceName);
        };
        Bootstrap.TASK.scheduleAtFixedRate(autoRequest,3, 5, TimeUnit.SECONDS);
    }

    /**
     * 获取初始化属性数据
     * @return 属性数据
     */
    private Map<String, ValueWrapper> getInitProperty(){
        properties = new HashMap<>();
        // SECTION_A
        properties.put(getPropName(DATA_NAME.IA), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.IB), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.IC), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.UAN), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.UBN), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.UCN), new ValueWrapper.DoubleValueWrapper(0.0));
        // SECTION_B
        properties.put(getPropName(DATA_NAME.W), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.VAR), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.VA), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.PF), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.FREQ), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.WA), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.WB), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.WC), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.VARA), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.VARB), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.VARC), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.VAA), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.VAB), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.VAC), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.PFA), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.PFB), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.PFC), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.KWH), new ValueWrapper.DoubleValueWrapper(0.0));
        // SECTION_C
        properties.put(getPropName(DATA_NAME.KVARH), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.KWH_J), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.KVARH_J), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.KWH_F), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.KVARH_F), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.KWH_P), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.KVARH_P), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.KWH_G), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME.KVARH_G), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME._KWH_HEX), new ValueWrapper.DoubleValueWrapper(0.0));
        properties.put(getPropName(DATA_NAME._KVARH_HEX), new ValueWrapper.DoubleValueWrapper(0.0));
        return properties;
    }

    /**
     * 更新片段A 的属性数据
     * @param payload 载荷
     */
    private void processSectionA(byte[] payload){
        // 读取 前3个 2字节寄存器
        for (int i = 0; i < 3; i++) {
            DATA_NAME dataName = DATA_NAME.values()[i];
            double value = byteToInt16(payload[i*2], payload[i*2+1]) * 1.0 / dataName.getDecimal();
            properties.put(getPropName(dataName), new ValueWrapper.DoubleValueWrapper(value));
        }
        // 读取 后3个 2字节寄存器
        for (int i = 4; i < 7; i++) {
            DATA_NAME dataName = DATA_NAME.values()[i-1];
            double value = byteToInt16(payload[i*2], payload[i*2+1]) * 1.0 / dataName.getDecimal();
            properties.put(getPropName(dataName), new ValueWrapper.DoubleValueWrapper(value));
        }
    }

    /**
     * 更新片段B 的属性数据
     * @param payload 载荷
     */
    private void processSectionB(byte[] payload){
        // 读取 前17个 2字节寄存器
        for (int i = 0; i < 17; i++) {
            DATA_NAME dataName = DATA_NAME.values()[i + DATA_NAME.W.ordinal()];
            double value = byteToInt16(payload[i*2], payload[i*2+1]) * 1.0 / dataName.getDecimal();
            properties.put(getPropName(dataName), new ValueWrapper.DoubleValueWrapper(value));
        }
        // 读取 第18个 4字节寄存器
        double value = (byteToInt16(payload[34], payload[35]) * 65536 + byteToInt16(payload[36], payload[37]))
                * 1.0 / DATA_NAME.KWH.getDecimal();
        properties.put(getPropName(DATA_NAME.KWH), new ValueWrapper.DoubleValueWrapper(value));
    }

    /**
     * 更新片段C 的属性数据
     * @param payload 载荷
     */
    private void processSectionC(byte[] payload){
        // 读取 供11个 4字节寄存器
        for (int i = 0; i < 11; i++) {
            DATA_NAME dataName = DATA_NAME.values()[i + DATA_NAME.KVARH.ordinal()];
            double value = (byteToInt16(payload[i*4], payload[i*4+1]) * 65536 + byteToInt16(payload[i*4+2], payload[i*4+3]))
                    * 1.0 / DATA_NAME.KWH.getDecimal();
            properties.put(getPropName(dataName), new ValueWrapper.DoubleValueWrapper(value));
        }
    }

    /**
     * 设备的初始化参数
     * @return Map<String, ValueWrapper>
     */
    @Override
    public Map<String, ValueWrapper> initProperty() {
        return properties;
    }

    /**
     * 设备当前的参数
     * @return Map<String, ValueWrapper>
     */
    @Override
    public Map<String, ValueWrapper> getProperty() {
        return properties;
    }

    /**
     * 设置设备的参数
     *
     * @param key    参数名称
     * @param value  参数内容
     */
    @Override
    public void setProperty(String key, Object value) {
        // 该设备不支持属性设置
        throw new SystemRuntimeException("设备:{} 不支持设置参数 {}:{}", deviceName, key, value);
    }


    /**
     * 获取设备名称
     * @return 设备名称
     */
    @Override
    public DeviceName getName() {
        return this.deviceName;
    }

    /**
     * 获取设备类型
     * @return 设备类型
     */
    @Override
    public DeviceType getType() {
        return DeviceType.SERIAL_PDM803DL;
    }
}
```