# 00_network_fundamentals

## Cel
Zrozumienie podstaw adresacji IP, modelu OSI oraz najważniejszych protokołów i portów sieciowych z perspektywy analityka SOC. 

## Zakres
- `ip_addressing_and_subnets` - adresacja IPv4, maski podsieci, CIDR, obliczanie hostów, adresy publiczne i prywatne, gateway, subnetting, 
- `osi_model_for_soc` - model OSI (7 warstw) i model TCP/IP (4 warstwy), zadania poszczególnych warstw, protokoły i zagrożenia z perspektywy SOC,
- `common_ports_and_protocols` - przegląd protokołów sieciowych (HTTP, DNS, SSH, TCP, UDP, ARP i innych), wraz z portami, sposobem działania i zagrożeniami. 

## Dlaczego to ważne 
- `Adresacja IP` - podstawa analizy logów, wykrywania skanowania sieci i identyfikacji źródła ataku, 
- `Model OSI` - wspólny język do opisywania problemów sieciowych i ataków (np. atak na warstwę 2 = ARP spoofing),
- `Protokoły i porty` - znajomość standardowych i podejrzanych portów umożliwia szybkie wykrycie reverse shell, C2 i eksfiltracji danych.