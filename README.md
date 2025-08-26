conn mikrotik

auto=start

keyexchange=ikev2

ike=aes256-sha256-modp2048

esp=aes256-sha256-modp2048

left=%defaultroute

leftauth=psk

leftsourceip=%config

right=10.78.17.54 #ippublic dari mikrotik

rightauth=psk

rightsubnet=10.10.10.0/24 #subnetyangdidapatkanolehkalinantinya

type=tunnel
