
# MongooseIM 与 FreeSWITCH 的消息集成

## MongooseIM 配置

**添加 FreeSWITCH 作为外部 XMPP 组件：**  
在 MongooseIM 的配置文件 `ejabberd.cfg` 或 `mongooseim.cfg` 中，添加 FreeSWITCH 作为外部组件。

```yaml
listen:
  - port: 5222
    module: ejabberd_c2s
  - port: 5269
    module: ejabberd_s2s_in
  - port: 5275
    module: ejabberd_service
    access: all
    shaper_rule: fast
    hosts:
      "freeswitch.domain.com":
        password: "your_secret_password"
  - port: 5223
    module: ejabberd_c2s
    access: all
    shaper_rule: fast
    tls: true
```

## FreeSWITCH 配置

**配置 mod_dingaling：**  
在 FreeSWITCH 的配置目录 `conf/autoload_configs/` 中，找到 `dingaling.conf.xml` 文件，配置连接到 MongooseIM 的参数。

```xml
<configuration name="dingaling.conf" description="Dingaling Configuration">
  <settings>
    <!-- 全局设置 -->
    <param name="debug" value="1"/>
    <param name="codec-prefs" value="$${global_codec_prefs}"/>

    <!-- SSL/TLS 设置 -->
    <param name="use-tls" value="true"/>

    <!-- Jingle 到 SIP 网关设置 -->
    <param name="context" value="public"/>
    <param name="rtp-ip" value="$${local_ip_v4}"/>
    <param name="ext-rtp-ip" value="auto-nat"/>
    <param name="sip-ip" value="$${local_ip_v4}"/>
    <param name="ext-sip-ip" value="auto-nat"/>
  </settings>

  <gateways>
    <!-- 配置连接到 MongooseIM 的网关 -->
    <gateway name="xmpp_gateway">
      <param name="username" value="YOUR_USERNAME@YOUR_XMPP_SERVER"/>
      <param name="password" value="YOUR_PASSWORD"/>
      <param name="dialplan" value="XML"/>
      <param name="profile" value="internal"/>
      <param name="server" value="YOUR_XMPP_SERVER_IP_OR_HOSTNAME"/>
      <param name="exten" value="1234"/>
    </gateway>
  </gateways>

  <profiles>
    <!-- Google Talk 的示例配置 -->
    <profile name="google-talk">
      <param name="login" value="YOUR_USERNAME@gmail.com"/>
      <param name="password" value="YOUR_PASSWORD"/>
      <param name="dialplan" value="XML"/>
      <param name="context" value="default"/>
      <param name="rtp-ip" value="$${local_ip_v4}"/>
      <param name="sasl" value="plain"/>
    </profile>
  </profiles>
  
  <!-- 动态包含 jingle_profiles 目录中的所有 XML 配置文件 -->
  <X-PRE-PROCESS cmd="include" data="../jingle_profiles/*.xml"/>
</configuration>
```

## 确认 settings 和 gateways 的用途
- **Settings（设置）部分**：通常用于配置全局参数，这些参数影响 FreeSWITCH 的整体行为和连接选项。
- **Gateways（网关）部分**：用于配置 FreeSWITCH 如何作为客户端连接到外部 XMPP 服务器或服务。

在 `mod_dingaling` 的配置中：
- **Settings**：设置全局的调试选项、编解码器偏好、网络参数等。
- **Gateways**：配置 FreeSWITCH 连接到外部 XMPP 服务器，如 MongooseIM。这定义了 FreeSWITCH 如何作为 XMPP 客户端连接到外部服务。

## 双向消息传递的工作原理

### MongooseIM 到 FreeSWITCH

1. 当 MongooseIM 用户向 FreeSWITCH 用户发送消息时，消息通过 XMPP 协议传递给 FreeSWITCH。
2. FreeSWITCH 根据 chatplan 的配置将消息路由给相应的 SIP 用户。

### FreeSWITCH 到 MongooseIM

1. 当 FreeSWITCH 用户向 MongooseIM 用户发送消息时，消息通过 SIP 协议传递给 FreeSWITCH。
2. FreeSWITCH 使用 mod_dingaling 模块将 SIP 消息转换为 XMPP 消息，并发送给 MongooseIM 用户。

## 具体配置步骤

### 在 FreeSWITCH 中配置 chatplan

确保 `chatplan.conf.xml` 中包含相应的路由规则，将 XMPP 消息路由给正确的 SIP 用户。

```xml
<configuration name="chatplan.conf" description="Chatplan Configuration">
  <profiles>
    <profile name="default">
      <mappings>
        <map from="dingaling" to="^.*$" action="message" data="sofia/internal/${destination_number}@sipdomain.com"/>
      </mappings>
    </profile>
  </profiles>
</configuration>
```

### 在 MongooseIM 中配置路由规则

确保 MongooseIM 配置文件中包含正确的路由规则，将消息发送给 FreeSWITCH 的用户。

```yaml
listen:
  - port: 5222
    module: ejabberd_c2s
  - port: 5269
    module: ejabberd_s2s_in
  - port: 5275
    module: ejabberd_service
    access: all
    shaper_rule: fast
    hosts:
      "freeswitch.domain.com":
        password: "your_secret_password"
```

## 注意事项
1. **替换占位符**：确保将所有占位符（如 `YOUR_XMPP_SERVER_IP_OR_HOSTNAME`、`YOUR_USERNAME@YOUR_XMPP_SERVER`、`YOUR_PASSWORD` 等）替换为实际值。
2. **路径和权限**：确认配置文件路径和 FreeSWITCH 的文件权限配置正确。
3. **重启 FreeSWITCH**：每次修改配置文件后，重启 FreeSWITCH 以应用更改：

```bash
service freeswitch restart
```

或者：

```bash
/usr/local/freeswitch/bin/freeswitch -rx 'reloadxml'
```

## 测试

1. **从 MongooseIM 用户发送消息到 FreeSWITCH 用户：**
   - 使用 XMPP 客户端登录 MongooseIM，发送消息到 FreeSWITCH 用户，观察消息是否被成功路由。

2. **从 FreeSWITCH 用户发送消息到 MongooseIM 用户：**
   - 使用 SIP 客户端登录 FreeSWITCH，发送消息到 MongooseIM 用户，观察消息是否被成功路由。

通过这些配置，MongooseIM 和 FreeSWITCH 的用户可以双向互相发送消息，实现互通。
