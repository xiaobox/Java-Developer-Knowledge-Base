# CSRF

CSRF(cross site request forgery) 跨站请求伪造。

攻击者盗用了你的身份，以你的名义发送恶意请求。CSRF能够做的事情包括：以你名义发送邮件，发消息，盗取你的账号，甚至于购买商品，虚拟货币转账......造成的问题包括：个人隐私泄露以及财产安全

防御： 将cookie设置为httponly
