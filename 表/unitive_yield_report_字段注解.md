# unitive_yield_report 表字段注解

> 表用途：半导体封装测试良率汇总报表

---

## 基础信息

| 字段名 | 数据类型 | 默认值 | 注释说明 |
|--------|----------|--------|----------|
| `id` | int(11) | NOT NULL AUTO_INCREMENT | 主键ID，自增 |
| `factory_name` | varchar(10) | NULL | 工厂名称 |
| `prod_name` | varchar(200) | NULL | 规格型号（产品名称） |
| `cus_name` | varchar(100) | NULL | 业务伙伴料号（客户料号） |
| `status` | int(11) | NULL | 数据状态：-1=删除，0=初始，1=有效 |
| `opn` | varchar(100) | NULL | OPN号（产品编号/料号） |
| `order_no` | varchar(100) | NULL | 订单号 |
| `work_order_no` | varchar(100) | NULL | 工单号 |
| `package_lot` | varchar(100) | NULL | 封装批次号 |
| `wafer_lot` | varchar(255) | NULL | 晶圆批次号 |
| `dc` | varchar(20) | NULL | DC（Date Code，生产日期代码） |
| `date` | varchar(100) | NULL | 日期 |
| `package_type` | varchar(50) | NULL | 封装形式（如TO-220、SOT-23等） |
| `statis_date` | varchar(50) | NULL | 统计月份 |
| `file_name` | varchar(50) | NULL | 导入文件名 |
| `yield_flag_version` | varchar(25) | NULL | 规则版本号 |
| `yield_flag` | varchar(10) | NULL | 档位（良率分档标识） |
| `report_type` | tinyint(4) | '0' | 报告类型：0=常温报告，1=高温报告 |
| `auto_flag` | int(11) | '0' | 自动导入标记：1=自动导入，0=手动导入 |
| `excel_index` | int(11) | NULL | Excel导入行号索引 |
| `detail_id` | varchar(255) | NULL | 明细ID（关联明细表） |
| `exceed` | varchar(150) | NULL | 超标说明/超标项 |
| `qty` | int(11) | NULL | 数量 |
| `file_md5` | varchar(255) | NULL | 文件MD5校验码（用于去重校验） |
| `template_id` | varchar(50) | NULL | 模板ID（导入模板标识） |
| `create_time` | datetime | CURRENT_TIMESTAMP | 创建时间 |
| `update_time` | datetime | CURRENT_TIMESTAMP ON UPDATE | 更新时间 |

## 常温测试不良数

| 字段名 | 数据类型 | 默认值 | 注释说明 |
|--------|----------|--------|----------|
| `sx` | int(11) | '0' | SX不良数（失效项） |
| `qd` | int(11) | '0' | QD不良数（失效项） |
| `hd` | int(11) | '0' | HD不良数（失效项） |
| `os` | int(11) | '0' | OS不良数（Open/Short，开短路失效） |
| `bdc` | int(11) | '0' | BDC不良数（失效项） |
| `cont` | int(11) | '0' | CONT不良数（接触不良） |
| `eas` | int(11) | '0' | EAS不良数（单脉冲雪崩能量失效） |
| `dvds` | int(11) | '0' | DVDS不良数（漏源电压差失效） |
| `bad` | int(11) | '0' | 其他不良数（Other Bad） |
| `bv` | int(11) | '0' | BV不良数（击穿电压失效，BVDSS） |
| `vth` | int(11) | '0' | VTH不良数（阈值电压失效） |
| `rdson` | int(11) | '0' | RDSON不良数（导通电阻失效） |
| `rg` | int(11) | '0' | RG不良数（栅极电阻失效） |
| `idss` | int(11) | '0' | IDSS不良数（漏源截止电流失效） |
| `bad_elect` | int(11) | '0' | 电性不良数 |
| `vfsd` | int(11) | '0' | VFSD不良数（正向压降失效） |
| `reject` | int(11) | '0' | REJECT不良数（拒收数） |
| `gmp` | int(11) | '0' | GMP不良数 |
| `igssisgs` | int(11) | '0' | IGSS/ISGS不良数（栅源漏电流失效） |
| `idon` | int(11) | '0' | IDON不良数（导通电流失效） |
| `insulation` | int(11) | '0' | 绝缘不良数 |
| `optical` | int(11) | '0' | 光检不良数 |
| `rbsoa` | int(11) | '0' | RBSOA不良数（反向偏置安全工作区失效） |
| `bin2` | int(11) | NULL | BIN2不良数 |
| `sw` | int(11) | '0' | SW不良数（开关时间失效） |
| `trr` | int(11) | '0' | TRR不良数（反向恢复时间失效） |
| `sc` | int(11) | '0' | SC不良数（短路失效） |
| `sx_num` | int(11) | '0' | SX数量 |
| `extest_num` | int(11) | '0' | 外检不良数 |
| `bin_10` | int(11) | '0' | 常温BIN10不良数（华羿专用） |
| `dc_others` | int(11) | '0' | 常温DC Others不良数（华羿专用） |

## 封装工序不良数

| 字段名 | 数据类型 | 默认值 | 注释说明 |
|--------|----------|--------|----------|
| `scribing` | int(11) | '0' | 划片不良数 |
| `adjust` | int(11) | '0' | 调准不良数 |
| `mount` | int(11) | '0' | 装片不良数 |
| `bond` | int(11) | '0' | 键合不良数 |
| `mold` | int(11) | '0' | 塑封不良数 |
| `eleplate` | int(11) | '0' | 电镀不良数 |
| `print` | int(11) | '0' | 打印不良数 |
| `cut` | int(11) | '0' | 切筋不良数 |
| `paramtest` | int(11) | '0' | 参数测试不良数 |
| `extest` | int(11) | '0' | 外检不良数 |
| `pack` | int(11) | '0' | 包装不良数 |
| `bad_package_num` | int(11) | '0' | 封装不良数 |
| `bad_num` | int(11) | '0' | 不良总数 |
| `cut_in` | int(11) | '0' | 切筋投入数 |

## 常温产出

| 字段名 | 数据类型 | 默认值 | 注释说明 |
|--------|----------|--------|----------|
| `assem_input` | int(11) | '0' | 封装投入数 |
| `assem_out` | int(11) | '0' | 封装产出数 |
| `test_out` | int(11) | '0' | 测试产出数 |
| `test_count` | int(11) | '0' | 测试次数 |
| `ordinary_test_out` | int(11) | '0' | 常温测试产出数 |
| `changes` | int(11) | '0' | 零头汇总（零散数量汇总） |
| `test_remnants` | int(11) | '0' | 测试零头（测试剩余零散数量） |
| `packing_remnants` | int(11) | '0' | 包装零头（包装剩余零散数量） |

## 常温不良率

| 字段名 | 数据类型 | 默认值 | 注释说明 |
|--------|----------|--------|----------|
| `os_rate` | decimal(6,5) | '0.00000' | OS不良率（开短路失效率） |
| `eas_rate` | decimal(6,5) | '0.00000' | EAS不良率（单脉冲雪崩能量失效率） |
| `dvds_rate` | decimal(6,5) | '0.00000' | DVDS不良率（漏源电压差失效率） |
| `bv_rate` | decimal(6,5) | '0.00000' | BV不良率（击穿电压失效率） |
| `vth_rate` | decimal(6,5) | '0.00000' | VTH不良率（阈值电压失效率） |
| `idss_rate` | decimal(6,5) | '0.00000' | IDSS不良率（漏源截止电流失效率） |
| `igss_rate` | decimal(6,5) | '0.00000' | IGSS不良率（栅源漏电流失效率） |
| `dc_rate` | decimal(6,5) | '0.00000' | DC不良率 |
| `rg_rate` | decimal(6,5) | '0.00000' | RG不良率（栅极电阻失效率） |
| `rdson_rate` | decimal(6,5) | '0.00000' | RDSON不良率（导通电阻失效率） |
| `package_rate` | decimal(6,5) | '0.00000' | 封装良率 |
| `test_rate` | decimal(6,5) | '0.00000' | 测试良率 |
| `bad_rate` | decimal(6,5) | '0.00000' | 其他不良率（Other Bad Rate） |
| `cont_rate` | decimal(6,5) | '0.00000' | CONT不良率（接触不良率） |
| `total_rate` | decimal(6,5) | '0.00000' | 总良率 |
| `rbsoa_rate` | decimal(6,5) | '0.00000' | RBSOA不良率（反向偏置安全工作区失效率） |
| `vfsd_rate` | decimal(6,5) | '0.00000' | VFSD不良率（正向压降失效率） |
| `reject_rate` | decimal(6,5) | '0.00000' | REJECT不良率（拒收率） |
| `idon_rate` | decimal(6,5) | '0.00000' | IDON不良率（导通电流失效率） |
| `gmp_rate` | decimal(6,5) | '0.00000' | GMP不良率 |
| `insulation_rate` | decimal(6,5) | '0.00000' | 绝缘不良率 |
| `light_rate` | decimal(6,5) | '0.00000' | 光检不良率 |
| `sw_rate` | decimal(6,5) | '0.00000' | SW不良率（开关时间失效率） |
| `trr_rate` | decimal(6,5) | '0.00000' | TRR不良率（反向恢复时间失效率） |
| `sc_rate` | decimal(6,5) | '0.00000' | SC不良率（短路失效率） |
| `ordinary_yield` | decimal(8,5) | '0.00000' | 常温良率 |
| `bin_10_rate` | decimal(6,5) | '0.00000' | 常温BIN10不良率（华羿专用） |
| `dc_others_rate` | decimal(6,5) | '0.00000' | 常温DC Others失效率（华羿专用） |

## 高温测试不良数

| 字段名 | 数据类型 | 默认值 | 注释说明 |
|--------|----------|--------|----------|
| `h_dc` | int(11) | '0' | 高温DC不良数 |
| `h_cont` | int(11) | '0' | 高温CONT不良数（高温接触不良） |
| `h_idss` | int(11) | '0' | 高温IDSS不良数（高温漏源截止电流失效） |
| `h_bvdss` | int(11) | '0' | 高温BVDSS不良数（高温击穿电压失效） |
| `h_vth` | int(11) | '0' | 高温VTH不良数（高温阈值电压失效） |
| `h_vfsd` | int(11) | '0' | 高温VFSD不良数（高温正向压降失效） |
| `h_rdson` | int(11) | '0' | 高温RDSON不良数（高温导通电阻失效） |
| `h_idon` | int(11) | '0' | 高温IDON不良数（高温导通电流失效） |
| `h_eas` | int(11) | '0' | 高温EAS不良数（高温单脉冲雪崩能量失效） |
| `h_rg` | int(11) | '0' | 高温RG不良数（高温栅极电阻失效） |
| `h_dvds` | int(11) | '0' | 高温DVDS不良数（高温漏源电压差失效） |
| `h_os` | int(11) | '0' | 高温OS不良数（高温开短路失效） |
| `h_gmp` | int(11) | '0' | 高温GMP不良数 |
| `h_rbsoa` | int(11) | '0' | 高温RBSOA不良数（高温反向偏置安全工作区失效） |
| `h_igssisgs` | int(11) | '0' | 高温IGSS/ISGS不良数（高温栅源漏电流失效） |
| `h_trr` | int(11) | '0' | 高温TRR不良数（高温反向恢复时间失效） |
| `h_sw` | int(11) | '0' | 高温SW不良数（高温开关时间失效） |
| `h_sc` | int(11) | '0' | 高温SC不良数（高温短路失效） |
| `h_insulation` | int(11) | '0' | 高温绝缘不良数 |
| `h_optical` | int(11) | '0' | 高温光检不良数 |
| `h_other` | int(11) | '0' | 高温其他不良数（高温光检不良） |
| `h_bin_10` | int(11) | '0' | 高温BIN10不良数（华羿专用） |
| `h_dc_others` | int(11) | '0' | 高温DC Others不良数（华羿专用） |

## 高温产出与良率

| 字段名 | 数据类型 | 默认值 | 注释说明 |
|--------|----------|--------|----------|
| `h_output` | int(11) | '0' | 高温产出数 |
| `h_yield` | decimal(8,5) | '0.00000' | 高温良率 |
| `h_other_rate` | decimal(6,5) | '0.00000' | 高温其他不良率 |
| `h_bin_10_rate` | decimal(6,5) | '0.00000' | 高温BIN10不良率（华羿专用） |
| `h_dc_others_rate` | decimal(6,5) | '0.00000' | 高温DC Others失效率（华羿专用） |

## 高温不良率

| 字段名 | 数据类型 | 默认值 | 注释说明 |
|--------|----------|--------|----------|
| `h_dc_rate` | decimal(6,5) | '0.00000' | 高温DC不良率 |
| `h_cont_rate` | decimal(6,5) | '0.00000' | 高温CONT不良率 |
| `h_idss_rate` | decimal(6,5) | '0.00000' | 高温IDSS不良率 |
| `h_bvdss_rate` | decimal(6,5) | '0.00000' | 高温BVDSS不良率 |
| `h_vth_rate` | decimal(6,5) | '0.00000' | 高温VTH不良率 |
| `h_vfsd_rate` | decimal(6,5) | '0.00000' | 高温VFSD不良率 |
| `h_rdson_rate` | decimal(6,5) | '0.00000' | 高温RDSON不良率 |
| `h_idon_rate` | decimal(6,5) | '0.00000' | 高温IDON不良率 |
| `h_eas_rate` | decimal(6,5) | '0.00000' | 高温EAS不良率 |
| `h_rg_rate` | decimal(6,5) | '0.00000' | 高温RG不良率 |
| `h_dvds_rate` | decimal(6,5) | '0.00000' | 高温DVDS不良率 |
| `h_os_rate` | decimal(6,5) | '0.00000' | 高温OS不良率 |
| `h_gmp_rate` | decimal(6,5) | '0.00000' | 高温GMP不良率 |
| `h_rbsoa_rate` | decimal(6,5) | '0.00000' | 高温RBSOA不良率 |
| `h_igssisgs_rate` | decimal(6,5) | '0.00000' | 高温IGSS/ISGS不良率 |
| `h_trr_rate` | decimal(6,5) | '0.00000' | 高温TRR不良率 |
| `h_sw_rate` | decimal(6,5) | '0.00000' | 高温SW不良率 |
| `h_sc_rate` | decimal(6,5) | '0.00000' | 高温SC不良率 |
| `h_insulation_rate` | decimal(6,5) | '0.00000' | 高温绝缘不良率 |
| `h_optical_rate` | decimal(6,5) | '0.00000' | 高温光检不良率 |

## 分档信息

| 字段名 | 数据类型 | 默认值 | 注释说明 |
|--------|----------|--------|----------|
| `prod_names` | varchar(255) | NULL | 分档产品名（多个用逗号隔开） |
| `opns` | varchar(255) | NULL | 分档OPN（多个用逗号隔开） |

## 复测信息

| 字段名 | 数据类型 | 默认值 | 注释说明 |
|--------|----------|--------|----------|
| `retest_order_no` | varchar(100) | NULL | 复测单号（以'-F'结尾） |

---

## 术语说明

| 缩写 | 全称 | 说明 |
|------|------|------|
| OS | Open/Short | 开短路测试 |
| EAS | Single Pulse Avalanche Energy | 单脉冲雪崩能量 |
| DVDS | Drain-Source Voltage Difference | 漏源电压差 |
| BV / BVDSS | Breakdown Voltage (Drain-Source) | 漏源击穿电压 |
| VTH | Threshold Voltage | 阈值电压 |
| RDSON | On-Resistance (Drain-Source) | 导通电阻 |
| IDSS | Zero Gate Voltage Drain Current | 漏源截止电流 |
| IGSS | Gate Leakage Current | 栅源漏电流 |
| ISGS | Source-Gate Leakage Current | 源栅漏电流 |
| VFSD | Forward Voltage (Source-Drain) | 正向压降 |
| IDON | On-State Drain Current | 导通电流 |
| RBSOA | Reverse Bias Safe Operating Area | 反向偏置安全工作区 |
| SW | Switching Time | 开关时间 |
| TRR | Reverse Recovery Time | 反向恢复时间 |
| SC | Short Circuit | 短路 |
| GMP | - | 特定失效项 |
| DC | Date Code | 生产日期代码 |
| OPN | Ordering Part Number | 订购料号 |
| H- | High Temperature | 高温测试前缀 |

---

## 索引信息

| 索引名 | 类型 | 字段 | 说明 |
|--------|------|------|------|
| PRIMARY | 主键 | `id` | 自增主键 |
| opn_dc_indx | 普通索引 | `opn`, `dc` | 按OPN和DC查询 |
| lot_indx | 普通索引 | `package_lot` | 按封装批次查询 |