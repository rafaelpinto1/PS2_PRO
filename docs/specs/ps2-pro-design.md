# PS2 PRO: anotações do projeto

Reengenharia de hardware/firmware de um PS2 Slim, transformando ele num console "premium": SSD interno no lugar do memory card e da unidade óptica, HDMI nativo, WiFi/Bluetooth via ESP32, tela de status, RGB e um tema de OPL inspirado no PS5.

A ideia é documentar e publicar tudo (hardware + firmware) pra quem quiser replicar.

Objetivo, resumindo:

- Sem unidade óptica: jogos rodam do SSD interno
- SSD interno pra jogos e saves, sem memory card físico
- HDMI nativo, sem adaptador externo
- WiFi (rede/transferência) e Bluetooth (controles) via ESP32
- Telinha de status no topo do case
- RGB que reage ao jogo rodando
- Tema OPL customizado, inspirado no PS5

## O que já foi decidido

**A base é um PS2 Slim SCPH-90006, motherboard GH-072-42**, região Hong Kong, confirmado direto na placa. É a última revisão da linha Slim: EE, RDRAM, SPU2 e IOP unificados num ASIC só, com o GS de novo como chip separado (diferente do "Zeus" das 7000x, que integra EE+GS). Essa revisão também trouxe de volta a fonte interna (as 7000x/7900x usam fonte externa), o que muda o espaço disponível dentro do case (ver riscos mais abaixo).

Padrão de vídeo é NTSC-J.

E o achado mais importante até agora: **a placa já tem um modchip Matrix Infinity 1.9 soldado**. Ele tem um modo de boot chamado DEV2 que carrega automaticamente `hdd0:/__boot/BOOT.ELF` direto de um HD/SSD via ATA/IDE de verdade. Só que essa via exige solda no ponto de IDE da placa, e no caso da nossa GH-072-42 esse ponto é um cabo flex (FPC), praticamente inviável de soldar sem risco real de estragar a trilha.

**Decisão: o SSD vai ser conectado via adaptador USB-SATA, aproveitando a porta USB que já existe (sem solda nenhuma)**. Motivo principal: o ponto de solda do IDE fica no cabo flex do leitor óptico, praticamente inviável de soldar sem risco real de estragar a trilha. Isso troca velocidade (USB 1.1 do PS2 é bem mais lento que IDE nativo, ~1-1.3 MB/s reais contra os ~66 MB/s teóricos do IDE) pela viabilidade prática de instalar sem risco de dano à placa. M.2 NVMe foi descartado (precisa de controlador PCIe que o PS2 não tem); rede/Ethernet embutida também (não resolve o boot sozinha e exigiria servidor SMB rodando em outro lugar).

**Plano de teste em andamento:**
1. Testar primeiro com um pendrive via USB (barato, rápido de testar, não compromete nada).
2. Se funcionar bem, seguir com SSD via adaptador USB-SATA (ou M.2 SATA + adaptador).
3. Se o USB 1.1 deixar o carregamento muito lento na prática, o plano B é testar via **MX4SIO** (porta de memory card), que é mais rápido que USB (~2-2.5 MB/s) e também não exige solda.

Como o modo DEV2 do Matrix Infinity só reconhece ATA/IDE, ele **não** dá boot automático de um SSD ligado por USB. A alternativa é o modo **"mass"** do próprio chip, que também funciona com pendrive/USB, só que em vez de bootar direto no BOOT.ELF, ele entra no uLaunchELF (um gerenciador de arquivos). O uLaunchELF tem uma opção de configurar um "Auto launch" apontando pro caminho do OPL, o que pode dar um boot automático (ou quase automático) mesmo sem o DEV2. Isso ainda precisa ser testado na prática (ver checklist).

## O que ainda tá em aberto

- **Boot automático via USB**: testar se o modo "mass" do Matrix Infinity + a opção de "Auto launch" do uLaunchELF conseguem dar boot direto no OPL sem precisar navegar manualmente toda vez que ligar o console.
- **Formato do SSD via USB**: como o acesso passa a ser via USB (modo `mass:`/`usb:` do OPL), não precisa mais de partição APA+PFS como precisaria via DEV2/IDE. O SSD deve poder ser formatado direto em FAT32 ou exFAT, como qualquer pendrive.

## Ficha técnica

| Campo | Valor |
|---|---|
| Base | PS2 Slim SCPH-90006, motherboard GH-072-42 (Hong Kong) |
| Armazenamento | SSD via adaptador USB-SATA (aproveitando a porta USB existente, sem solda). 100% dos jogos e saves, sem memory card |
| Boot | Modchip Matrix Infinity 1.9 (já instalado), modo "mass" + auto launch do uLaunchELF (a confirmar na prática) |
| Software de boot | OPL com tema customizado inspirado no PS5, carregado via USB |
| Vídeo | HDMI nativo, tap digital do GS (pré-DAC) + encoder HDMI embutido no case |
| Conectividade | ESP32 clássico (não o S3): WiFi 2.4GHz + Bluetooth Classic + BLE. Classic é obrigatório pro fone Bluetooth (A2DP) e pra parear controles PS4/PS5/Xbox originais (o S3 só tem BLE e não serve) |
| Entrada | Controles Bluetooth genéricos do mercado (PS4/PS5/Xbox/8BitDo), convertidos de HID pro protocolo da porta de controle do PS2 via ESP32. Sem controle físico próprio, ver comparativo com o PS5 mais abaixo |
| Display auxiliar | Tela OLED/LCD pequena no topo do case: rede, temperatura, storage, hora, capa/título do jogo rodando |
| Iluminação | LEDs endereçáveis refletindo a cor dominante do jogo (tipo o DualSense) |
| Rede | Servidor FTP no próprio ESP32, com acesso direto ao SSD. Não depende do software do PS2/OPL |
| Alimentação | Bateria interna recarregável (meta de 1-2h de autonomia, ainda ligado numa TV) + circuito de carga/descarga |
| Resfriamento | Revisar o fan/heatsink original considerando os componentes novos (SSD, ESP32, encoder HDMI, bateria). O case fica mais cheio e mais quente que o original |
| Depois da v1 | OTA de firmware, fonte USB-C PD, RTC com bateria própria, HDMI-CEC, app companion, painel web (modo AP), scraping de capas, BOM+case aberto, perfis de save, plugins de áudio, captura de tela/clipes, backup de saves na nuvem, RetroAchievements, SSD removível, upscaling/filtros de vídeo, acessibilidade, notificação de transferência, bateria do controle na tela, gerenciador de armazenamento, "jogados recentemente", skins extras |
| Licença | Aberto: hardware e firmware, BOM e case publicados |
| Fora do escopo | Netflix/Spotify, inviável no hardware do PS2 (ver mais abaixo) |

## Subsistemas

Divido em 5 partes, cada uma com uma versão inicial enxuta e uma lista de coisas pra depois:

| # | Subsistema | Versão inicial | Depois |
|---|---|---|---|
| 1 | Storage & Saves | SSD via USB-SATA + OPL configurado, boot via modo "mass" do Matrix Infinity; sem memory card, sem drive óptico | Migração de saves antigos; perfis multi-usuário; backup na nuvem; SSD removível/expansível (tipo slot M.2) |
| 2 | Vídeo HDMI | Tap digital do GS + encoder HDMI, saída fixa em 720p/1080p | HDMI-CEC; captura de tela/clipes; upscaling/filtros (CRT/sharpening) |
| 3 | Conectividade ESP32 | FTP direto no SSD; pareamento Bluetooth Classic (PS4/PS5/Xbox/8BitDo); tela de status; RGB por jogo | OTA; painel web; app companion; scraping de capas; remap de botões; fone Bluetooth (A2DP); notificação de transferência; bateria do controle na tela |
| 4 | Alimentação & Confiabilidade | Bateria interna recarregável + circuito de carga/descarga; revisar resfriamento (fan/heatsink) pros componentes novos | Fonte USB-C PD; RTC com bateria própria |
| 5 | Tema OPL | Tema visual inspirado no PS5 | Plugins de áudio; RetroAchievements; áudio mono (acessibilidade); gerenciador de armazenamento; "jogados recentemente"; skins extras |
| - | Acessibilidade (atravessa tudo) | - | Remap de botões (3); filtro de cor pra daltonismo no vídeo (2); áudio mono (5) |
| - | Comunidade (atravessa tudo) | - | BOM + case 3D publicados |

Backlog sem prioridade por enquanto: integração com Home Assistant/MQTT.

### Comparando com o PS5

Pra ter uma ideia de onde o PS2 PRO chega perto do PS5 e onde não dá:

| PS5 | No PS2 PRO | Dá? |
|---|---|---|
| DualSense (bateria, USB-C, RGB, haptics, giroscópio) | Controle Bluetooth genérico já existente, convertido pelo ESP32 | Sem controle customizado próprio no v1, isso seria um sub-projeto à parte |
| Áudio pela saída P2 do controle | Fone Bluetooth (A2DP) pareado direto no console | O ESP32 vira fonte A2DP; qualquer fone parear direto, com a latência de ~100-200ms típica do A2DP |
| SSD rápido / Quick Resume | SSD interno | SSD sim, quick resume não. O PS2 não tem esse mecanismo, é limitação de hardware/software mesmo |
| Capturas de tela e clipes | Reaproveita o tap do GS, salva no SSD | Dá, fica pra depois da v1 |
| Trophies | RetroAchievements | Dá, fica pra depois da v1 |
| Backup de saves na nuvem | Sync via WiFi pelo ESP32 | Dá, fica pra depois da v1 |
| Quick menu | App companion | Dá, já faz parte do plano do app companion |
| Remote Play | - | Não dá. O PS2 não codifica vídeo pra streaming, precisaria de hardware dedicado extra |
| Party/voice chat | - | Não dá, precisaria de uma stack de rede inteira nova |
| Áudio espacial | Plugins de áudio (upsampling/surround) | Já coberto, fica pra depois da v1 |
| Auto-update | OTA do ESP32 | Já no escopo |
| Netflix/Spotify | - | Não dá de jeito nenhum no PS2: o Emotion Engine (MIPS ~300MHz, ano 2000) não decodifica H.264/H.265 nem em software em tempo real, e a Netflix exige DRM (Widevine) com hardware certificado que o PS2 não tem. Só viraria viável com um SoC Android TV embutido à parte (avaliado e descartado por agora) |

### O risco do ESP32 fazer tudo sozinho

A ideia é um ESP32 só cuidando de WiFi, Bluetooth, display, RGB e config, mais simples e barato que separar em dois chips. O problema é que WiFi e Bluetooth dividem o mesmo rádio, o que pode gerar jitter/lag no controle durante tráfego de rede. E com o fone Bluetooth (A2DP) no meio, agora são três usos do mesmo rádio ao mesmo tempo (WiFi, BT do controle, BT do áudio), o que piora ainda mais a disputa.

Isso precisa ser medido na prática assim que tiver um protótipo: latência/jitter do controle e estabilidade do áudio A2DP com rede em uso simultâneo. Se não der pra aceitar o resultado, o plano B é separar num segundo chip dedicado (tipo um RP2040) só pro timing do controle e/ou do áudio.

## Diagrama de blocos

Ainda não tenho o esquema elétrico ponto-a-ponto (isso vai vir conforme eu for mapeando a placa física), só o diagrama funcional por enquanto:

```
[Placa-mãe PS2 Slim]
   ├─(sinal digital GS pré-DAC)──> [Encoder HDMI interno] ──> [Saída HDMI no case]
   ├─(porta USB existente)───────> [Adaptador USB-SATA] ──> [SSD] (jogos + saves, sem memory card)
   ├─(porta de controle x2)──────> [ESP32: conversor BT Classic→protocolo controle]
   └─(alimentação 3.3V/5V)───────> [ESP32 + periféricos]

[ESP32] ──I2C/SPI──> [Tela OLED/LCD no topo]
        ──RMT/GPIO─> [LEDs endereçáveis RGB]
        ──WiFi──────> [Servidor FTP próprio, app companion, painel web AP, OTA]
```

Como o SSD agora vai direto na porta USB do PS2 (não mais num barramento compartilhado tipo IDE), o ESP32 não tem mais acesso físico direto ao mesmo SSD. Isso muda a arquitetura do FTP/transferência de arquivos: precisa decidir se o ESP32 recebe um SSD/armazenamento próprio separado só pra isso, ou se existe algum jeito de compartilhar o mesmo SSD via um switch USB (ver riscos abaixo).

## Riscos e pendências

- Boot automático via USB ainda não testado na prática (modo "mass" + auto launch do uLaunchELF), ver seção acima.
- Velocidade de carregamento via USB 1.1 vai ser bem mais lenta que IDE nativo (~1-1.3 MB/s reais). Aceito como troca pela viabilidade de instalar sem risco à placa.
- Arquitetura de acesso do ESP32 ao SSD mudou: como o SSD vai na porta USB do PS2 (não num barramento compartilhado), o ESP32 não tem mais acesso direto ao mesmo storage. Precisa decidir entre o ESP32 ter seu próprio armazenamento separado (SD card, por exemplo) só pro FTP/transferência, ou algum esquema de switch USB compartilhando o mesmo SSD entre PS2 e ESP32.
- Jitter de WiFi/Bluetooth no controle, ver seção do ESP32 acima.
- Espaço interno do case: o Slim já é apertado, e o SCPH-90006 tem fonte interna ocupando parte do espaço (diferente das revisões com fonte externa). Encaixar encoder HDMI, ESP32, tela, LEDs e bateria dividindo espaço com a fonte já existente vai precisar de um mockup mecânico pra validar.
- Bateria interna: o PS2 Slim consome uns 25-35W, então pra 1-2h de autonomia precisa de algo como 30-60Wh (vários cells 18650 juntos), um pack grande pro espaço que sobra. Ainda falta o circuito de troca automática bateria/tomada, e preciso medir o consumo real antes de fechar a meta.
- Resfriamento: o fan/heatsink original foi pensado só pro EE/GS/IOP originais, sem os componentes novos (SSD, ESP32, encoder HDMI, circuito de carga da bateria). Isso esquenta mais o case do que o projeto original, e ainda por cima a bateria de Li-ion vai ficar perto dessas fontes de calor. Precisa validar se dá pra reaproveitar o fan original ou se precisa de algo maior, e pensar no posicionamento da bateria longe do que mais esquenta (GS, encoder, fonte interna).
  - Achado importante lendo o esquema (board scan do Mister M, ver `docs/schematics/`): a placa 900xx usa 7.5V como trilho principal (com um "LDO" próprio pro standby do mechacon), diferente da linha 7900x que usa 8.5V via MOSFET. No próprio tópico onde peguei o scan, o autor tentou alimentar os reguladores mapeados por fonte externa nessa mesma família 900xx e **não deu vídeo, só esquentou** (ele mesmo avisa pra não confiar cegamente nesse caminho). Só que outros usuários confirmaram numa 90001 que **bateria 2S (Li-ion/LiPo, 7.4V nominal, 8.4V cheia) funciona normal**, entrando bem próximo de onde a fonte interna já entrega os 7.5V. É bem diferente de tentar rebaixar pra 5V direto (um usuário queimou o MOSFET tentando isso). Ou seja: o caminho mais seguro pra bateria parece ser 2S entrando onde a fonte interna hoje entrega ~7.5V, deixando a cadeia de reguladores já existente na placa fazer o resto, não tentar alimentar cada trilho manualmente do zero.
- O tap digital do GS (pré-DAC) ainda não tá confirmado como acessível. Diferente do N64, que expõe um barramento de vídeo digital conhecido, não sei se o GS do PS2 expõe esse sinal em pinos externos antes de converter pra analógico. Essa é a base de todo o Subsistema 2 e precisa ser a primeira coisa a validar com a placa em mãos. Se não der, a alternativa é converter a partir da saída AV multi-out analógica.
- Upscaling/filtros de vídeo (CRT/sharpening) não rolam com um encoder HDMI simples, precisaria de um scaler tipo FPGA (OSSC/RetroTINK), bem mais caro e maior que um encoder básico. Isso entra no orçamento de espaço/custo do Subsistema 2.
- Falta definir o canal de dados PS2/OPL → ESP32 (pra ele saber qual jogo tá rodando, mostrar na tela, mudar o RGB, notificar transferência). Como o OPL é open-source, dá pra patchear ele pra emitir isso, é desenvolvimento de firmware de verdade, não só "tema visual".
- Arbitragem de acesso ao SSD entre ESP32 e PS2: como o ESP32 acessa direto pro FTP, precisa definir quando cada um pode acessar (ex: FTP só ativo com o PS2 desligado) pra não corromper dado.

## Material de apoio

- `img_videos/`: fotos e vídeo do console desmontado, estado inicial (antes de qualquer mod).
- `docs/schematics/`: board scan da GH-072-42 (mapa de tensões por cor, valores de componentes). Não é meu, peguei de https://bitbuilt.net/forums/threads/board-scan-ps2-90006-90010.4340/, créditos ao Mister M (Nexus-M). Ver o README daquela pasta pros detalhes de o que dá e o que não dá pra tirar de lá.
- `firmware/`: wLaunchELF e OPL (Open PS2 Loader) usados no teste de boot via pendrive. Homebrew de terceiros, não escrito por mim, créditos e licença de cada um no README da respectiva subpasta.

## Ordem de execução

Checklist pra ir acompanhando o que já foi feito/validado. Marcar conforme for avançando.

**Fase 0, validar antes de mexer**
- [x] Preparar o pendrive (FAT32, uLaunchELF na raiz, OPL numa pasta própria)
- [ ] Testar modo "mass" do Matrix Infinity com pendrive: uLaunchELF abre certinho (falta controle de PS2 físico pra fazer esse teste)
- [ ] Configurar "Auto launch" do uLaunchELF apontando pro OPL e testar se dá boot automático via USB
- [ ] Confirmar se o sinal digital do GS é acessível
- [ ] Mapear a fonte interna (rails, espaço, calor)

**Fase 1, Storage & Saves**
- [ ] Confirmar que o teste com pendrive funcionou antes de seguir pra instalação definitiva
- [ ] Soldar o adaptador USB-SATA direto nas trilhas internas da porta USB (instalação permanente, sem cabo pra fora)
- [ ] Formatar o SSD (FAT32 ou exFAT) e configurar o OPL
- [ ] Validar boot automático completo via USB (ou aceitar boot manual se o automático não funcionar)

**Fase 2, Vídeo HDMI**
- [ ] Prototipar o tap do GS + encoder HDMI em bancada

**Fase 3, ESP32**
- [ ] Prototipar pareamento de controle Bluetooth (PS4/PS5/Xbox/8BitDo) em bancada
- [ ] Prototipar a tela de status (topo do case: rede, temperatura, storage, hora, jogo atual)
- [ ] Prototipar o RGB reagindo ao jogo
- [ ] Prototipar WiFi + servidor FTP
- [ ] Validar jitter de WiFi/BT (controle travando durante tráfego de rede)
- [ ] Integrar tudo fisicamente na placa

**Fase 4, Alimentação & Confiabilidade**
- [ ] Medir consumo real com tudo instalado
- [ ] Dimensionar e instalar a bateria (2S)
- [ ] Revisar fan/heatsink pros componentes novos

**Fase 5, Tema OPL**
- [ ] Aplicar o tema visual (pode rodar em paralelo desde o início)

**Fase 6, Integração final**
- [ ] Case definitivo
- [ ] Publicar BOM + arquivos de case
