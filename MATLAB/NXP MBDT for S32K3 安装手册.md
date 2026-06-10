# NXP MBDT for S32K3 安装手册

## 一、依赖文件

- [armcortexm.mlpkginstall](https://ww2.mathworks.cn/matlabcentral/fileexchange/43095-embedded-coder-support-package-for-arm-cortex-m-processors)
- [SW32_MBDT_S32K3_1.8.0_D2512.mltbx](https://freescaleesd.flexnetoperations.com/337170/551/13877551/SW32_MBDT_S32K3_1.8.0_D2512.mltbx?ftpRequestID=4327419341&server=freescaleesd.flexnetoperations.com&dtm=DTM20260602102358MTYyMDE4OTQ5&authparam=1780421038_0481f4b408be48d738b994b6c83f8793&ext=.mltbx)

## 二、安装步骤

1. MATLAB 设置 -> 附加功能-> 安装文件夹，修改路径使其不带空格，中文和特殊符号
2. 使用 MATLAB 打开并安装 `armcortexm.mlpkginstall
3. 使用 MATLAB 打开并安装 `SW32_MBDT_S32K3_1.8.0_D2512.mltbx`
4. 执行 `mbd_s32k3_path()`
