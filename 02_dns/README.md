# 02_dns w kontekście SOC

Folder zawiera szczegółowe omówienie protokołu DNS z perspektywy analityka SOC / Blue Team.

## Cel
Zrozumienie, jak działa DNS, jakie zagrożenia z niego wynikają oraz jak wykrywać podejrzany ruch w logach. 

## Zakres
- `dns_basics` - hierarchia DNS, rekordy, strefy, transfery, DoS/DoT (DNS over HTTPS / DNS over TLS), narzędzia, 
- `dns_in_attacks` - ataki: DNS tunnelling, DGA, Fast Flux, cache poisoning, amplification, reconnaissance, NXDOMAIN attack,
- `dns_log_analysis` - case studies: analiza podejrzanego ruchu DNS i reakcja. 

## Dlaczego to ważne
DNS to fundament internetu - każde połączenie zaczyna się od zapytania DNS. Atakujący wykorzystują go do C2, eksfiltracji danych, DDoS i rozpoznania. Analityk SOC, który rozumie anomalie DNS, potrafi wykryć atak na wczesnym etapie. 