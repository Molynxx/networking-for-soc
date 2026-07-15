# 01_tcp_udp_basics
   
Folder zawiera szczegółowe omówienie dwóch najważniejszych protokołów transportowych `TCP` i `UDP` z perspektywy analityka SOC / Blue Team
  
## Cel 
Zrozumienie mechanizmów działania TCP (handshake, flagi, stany) i UDP (bezstanowość), rozpoznawanie ataków na warstwie transportowej oraz przesłanek do wyboru właściwego protokołu do detekcji. 

## Zakres
- `tcp_handshake_and_flags_PL.md` - Three-way handshake, flagi TCP (SYN, ACK, RST, FIN, PSH, URG), tcpdump, case study, ataki: SYN flood, TCP reset, session hijacking, 
- `udp-basics_PL.md` - Bezstanowość UDP, zalety i zagrożenia, case study, ataki: UDP flood, DNS amplification, NTP amplification,
- `tcp_vs_udp_in_attacks_PL.md` - Porównanie obu protokołów z perspektywy atakującego -> kiedy wybiera TCP, a kiedy UDP i dlaczego.  

## Dlaczego to ważne
TCP i UDP to fundament komunikacji sieciowej. Każdy atak - od DDoS po kradzież danych - używa jednego z tych protokołów. Analityk SOC musi wiedzieć, czego szukać: anomalie we flagach, podejrzany ruch UDP, nietypowe wzorce ruchu. Bez zrozumienia warstwy transportowej nie da się czytać logów ani wykrywać zagrożeń. 

