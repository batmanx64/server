# License 鉴权机制分析总结

## 概述

本文档总结了对 ONLYOFFICE Document Server 中 License 鉴权机制的完整分析，包括 Socket 连接处理、许可证验证链路以及如何创建有效的许可证文件。

## 1. Socket 连接中的 License 消息处理

### 1.1 消息监听器
```javascript
conn.on('message', data => {
  ctx.logger.info('data.type = %s', data.type);
  switch (data.type) {
    case 'license':
      // 处理 license 类型消息
      break;
    // ... 其他类型
  }
});
```

### 1.2 License 消息发送
在 `_checkLicense` 函数中发送许可证信息：
```javascript
sendData(ctx, conn, {
  type: 'license',
  license: {
    type: licenseInfo.type,
    mode: licenseInfo.mode,
    rights,
    buildVersion: commonDefines.buildVersion,
    buildNumber: commonDefines.buildNumber,
    protectionSupport: tenOpenProtectedFile,
    isAnonymousSupport: tenIsAnonymousSupport,
    liveViewerSupport: utils.isLiveViewerSupport(licenseInfo),
    branding: licenseInfo.branding,
    customization: licenseInfo.customization,
    advancedApi: licenseInfo.advancedApi
  },
  aiPluginSettings: pluginSettings
});
```

## 2. 许可证验证完整链路

### 2.1 核心方法：`tenantManager.getTenantLicense(ctx)`

**文件位置**: `Common/sources/tenantManager.js`

```javascript
async function getTenantLicense(ctx) {
  let res = licenseTuple;
  if (isMultitenantMode(ctx) && !isDefaultTenant(ctx)) {
    // 多租户模式下读取租户许可证
    const tenantPath = utils.removeIllegalCharacters(ctx.tenant);
    const licensePath = path.join(cfgTenantsBaseDir, tenantPath, cfgTenantsFilenameLicense);
    let licenseTupleTenant = nodeCache.get(licensePath);
    if (licenseTupleTenant) {
      ctx.logger.debug('getTenantLicense from cache');
    } else {
      licenseTupleTenant = await readLicenseTenant(ctx, licensePath, licenseInfo);
      fixTenantLicense(ctx, licenseInfo, licenseTupleTenant[0]);
      nodeCache.set(licensePath, licenseTupleTenant);
      ctx.logger.debug('getTenantLicense from %s', licensePath);
    }
    res = licenseTupleTenant;
  }
  return res;
}
```

### 2.2 租户许可证解析：`readLicenseTenant()`

**关键安全特性**：
- **不验证签名**：`delete oLicense['signature'];` - 租户许可证签名被删除不验证
- **日期验证**：严格检查 `startDate <= now <= endDate`
- **宽限期机制**：过期后可继续运行但限制连接数

```javascript
// 日期验证逻辑
const timeLimited = 0 !== (res.mode & c_LM.Limited);
const checkDate = res.mode & c_LM.Trial || timeLimited ? new Date() : licenseInfo.buildDate;

if (startDate <= checkStartDate && checkDate <= res.endDate) {
  res.type = c_LR.Success;
} else if (startDate > checkStartDate) {
  res.type = c_LR.NotBefore;
} else if (timeLimited) {
  // 宽限期逻辑
  if (res.endDate.setUTCDate(res.endDate.getUTCDate() + res.graceDays) >= checkDate) {
    res.type = c_LR.SuccessLimit;
    res.connections = Math.min(res.connections, constants.LICENSE_CONNECTIONS);
  } else {
    res.type = c_LR.ExpiredLimited;
  }
}
```

## 3. 许可证鉴权机制分析

### 3.1 主许可证验证
**结论**：开源版本中**没有找到**主许可证的实际签名验证代码。

- `Common/sources/license.js` 仅返回默认开源许可证
- 虽然代码中多次提到"verify main lic signature"，但实际验证逻辑在开源版本中被移除或简化
- 项目包含 `jsrsasign` 库，但未在许可证验证中使用

### 3.2 租户许可证验证
**明确不需要签名验证**：
```javascript
//do not verify tenant signature. verify main lic signature.
//delete from object to keep signature secret
delete oLicense['signature'];
```

### 3.3 许可证文件格式

**必需字段**：
```json
{
  "start_date": "YYYY-MM-DD",
  "end_date": "YYYY-MM-DD",
  "customer_id": "string",
  "alias": "string",
  "multitenancy": boolean
}
```

**可选字段**：
```json
{
  "trial": boolean,
  "timelimited": boolean,
  "developer": boolean,
  "branding": boolean,
  "customization": boolean,
  "advanced_api": boolean,
  "connections": number,
  "users_count": number,
  "grace_days": number,
  "signature": "signature_string"
}
```

## 4. 如何创建有效的许可证文件

### 4.1 基本要求
1. **格式**：JSON 文件
2. **位置**：多租户模式下位于 `{cfgTenantsBaseDir}/{tenantPath}/.license`
3. **编码**：UTF-8
4. **权限**：服务器进程可读

### 4.2 示例许可证文件
```json
{
  "start_date": "2024-01-01",
  "end_date": "2025-12-31",
  "customer_id": "CUSTOMER123",
  "alias": "mycompany",
  "multitenancy": true,
  "trial": false,
  "timelimited": true,
  "developer": false,
  "branding": true,
  "customization": true,
  "advanced_api": true,
  "connections": 100,
  "users_count": 50,
  "grace_days": 30
}
```

### 4.3 许可证模式位运算
```javascript
// 许可证模式计算
if (true === oLicense['timelimited']) {
  res.mode |= c_LM.Limited;  // 4
}
if (Object.hasOwn(oLicense, 'trial')) {
  res.mode |= true === oLicense['trial'] ? c_LM.Trial : c_LM.None;  // 1 或 0
}
if (true === oLicense['developer']) {
  res.mode |= c_LM.Developer;  // 2
}
```

## 5. 许可证结果状态

| 状态码 | 常量 | 说明 |
|--------|------|------|
| 3 | `Success` | 许可证有效 |
| 7 | `SuccessLimit` | 宽限期内有效但受限 |
| 2 | `Expired` | 许可证过期 |
| 6 | `ExpiredTrial` | 试用期过期 |
| 11 | `ExpiredLimited` | 时间限制许可证过期 |
| 16 | `NotBefore` | 许可证在开始日期前 |
| 1 | `Error` | 许可证错误 |

## 6. 安全机制总结

### 6.1 缓存机制
- **NodeCache**：TTL=600秒，避免频繁磁盘I/O
- **密钥缓存**：`pemfileCache` 缓存密钥文件

### 6.2 权限降级
- 许可证异常时自动降为 0 用户/连接
- 宽限期内限制连接数至 20

### 6.3 JWT 安全
- 使用 HS256/HS384/HS512 对称加密
- 密钥通过配置文件或文件读取
- 支持过期时间和算法配置

## 7. 结论

这个开源版本的代码似乎**不包含完整的商业许可证验证功能**。实际的许可证签名验证和生成逻辑可能在商业版本或企业版中实现。开源版本主要依赖于：

1. **字段格式验证**：JSON 结构和必需字段检查
2. **时间限制检查**：严格的日期范围验证
3. **宽限期机制**：过期后的缓冲期处理
4. **权限降级策略**：异常情况下的安全降级

要创建有效的许可证文件，只需要确保 JSON 格式正确、日期有效且包含必需字段即可通过开源版本的验证。