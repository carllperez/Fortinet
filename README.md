# Fortinet

LDAP

config user ldap
    edit "AD-Server"
        set server "10.10.10.10"
        set port 389
        set cnid "sAMAccountName"
        set dn "dc=yourdomain,dc=local"
        set type regular
        set username "LOCAL\Administrator"
        set password "C1sc0123"
    end
end
