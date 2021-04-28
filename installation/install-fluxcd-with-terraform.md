# Overview

本篇文章主要讲解如何使用terraform 安装Fluxv2



# 注意事项

使用Fluxv2 监控 Git Repository 作为源输入时：

1. Spec.URL 必须是`^(http|https|ssh)://`  pattern 
2. 如果是git ssh认证的secret 必须要包含`identity`, `identity.pub ` 和 `known_hosts`
3. 如果是git https认证的必须包含 `username` and `password`



使用Fluxv2监控Helm Repository 作为源输入时：

1. 如果是HTTP/S的认证就必须要包含username 跟password的 field 。 
2. 如果是TLS的secret 必须要包含certFile和keyFile，caCert是optional的





