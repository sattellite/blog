# Cisco не позволяет подключиться по SSH

Случилась проблема: на Cisco устройство невозможно подключиться через SSH.
Проблема особенна тем, что нельзя подключиться только с linux-версии OpenSSH (Gentoo пользователям повезло больше), а с bsd-версии все отлично.

Проблема:

    ssh -v host
    OpenSSH_6.4, OpenSSL 1.0.1e-fips 11 Feb 2013
    debug1: Reading configuration data /home/sattellite/.ssh/config
    debug1: /home/sattellite/.ssh/config line 60: Applying options for host
    debug1: Reading configuration data /etc/ssh/ssh_config
    debug1: /etc/ssh/ssh_config line 51: Applying options for *
    debug1: Connecting to host [host] port 22.
    debug1: Connection established.
    debug1: identity file /home/sattellite/.ssh/id_rsa type 1
    debug1: identity file /home/sattellite/.ssh/id_rsa-cert type -1
    debug1: identity file /home/sattellite/.ssh/id_dsa type -1
    debug1: identity file /home/sattellite/.ssh/id_dsa-cert type -1
    debug1: identity file /home/sattellite/.ssh/id_ecdsa type -1
    debug1: identity file /home/sattellite/.ssh/id_ecdsa-cert type -1
    debug1: Enabling compatibility mode for protocol 2.0
    debug1: Local version string SSH-2.0-OpenSSH_6.4
    debug1: Remote protocol version 2.0, remote software version Cisco-1.25
    debug1: no match: Cisco-1.25
    debug1: SSH2_MSG_KEXINIT sent
    debug1: SSH2_MSG_KEXINIT received
    debug1: kex: server->client aes128-cbc hmac-md5 none
    debug1: kex: client->server aes128-cbc hmac-md5 none
    debug1: SSH2_MSG_KEX_DH_GEX_REQUEST(1024<3072<8192) sent
    debug1: expecting SSH2_MSG_KEX_DH_GEX_GROUP
    Connection closed by host

Решение (найдено в [багзилле RedHat](https://bugzilla.redhat.com/show_bug.cgi?id=1026430)):

    ssh -o HostKeyAlgorithms=ssh-rsa,ssh-dss -o KexAlgorithms=diffie-hellman-group1-sha1 -o Ciphers=aes128-cbc,3des-cbc -o MACs=hmac-md5,hmac-sha1 -v host

Для пущего порядка можно записать все это в `.ssh/config`.


