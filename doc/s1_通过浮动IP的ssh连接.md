1. 设置openstack(该部分已完成)
    - 网络->安全组->default->添加规则->ssh
2. 本地操作:
    - Mac:
        - cd ~/.ssh 
        - vim k1_rsa #粘贴私钥
        - vim config #
            ```
            Host            centos-ext-k1
            HostName        202.120.40.8
            Port            30570
            User            centos
            IdentityFile    ~/.ssh/k1_rsa
            ```
        - ssh centos-ext-k1 #即可连接
        - 若遇到`Permissions 0644 for '/Users/bian/.ssh/k1_rsa' are too open.`
            - chmod 400 k1_rsa #即可
        
        - 其它未绑定浮动IP的server连接:
            - vim config
            ```
            Host            centos-ext-k3
            HostName        10.0.0.66
            Port            22
            User            centos
            ProxyCommand ssh centos-ext-k1 -W %h:%p
            IdentityFile    ~/.ssh/k3_rsa
            ```
