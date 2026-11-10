# Competição Rover — pacote das equipes

Este pacote mostra o que o robô e a ground station (GS) da equipe precisam fazer para funcionar com o dashboard da competição. As regras de interface estão em CONTRATO-EQUIPES.md; leia primeiro as seções 3 (MQTT) e 4 (vídeo).

---

## Fluxograma

ROBÔ (Raspberry Pi) ──vídeo H.264 UDP:5000──► GROUND STATION (notebook) ──Retransmitir (WHIP)──► SERVIDOR 192.168.3.2
   │  ▲                                              │                                            (dashboard, juízes)
   │  └──── comandos rover/<id>/cmd/* ◄──── broker MQTT 192.168.3.2:1883 ◄──── comandos da GS
   └──── telemetria rover/<id>/{status,state,mode,battery,detection} ────────────►
O que está aqui
Pasta / arquivo	Para quê
CONTRATO-EQUIPES.md	Obrigatório. Tópicos MQTT, padrão de vídeo, regras, homologação
cartoes/roverNN.md	ID e senha de cada equipe (o professor entrega só o da sua equipe)
referencia/rover-gs/	Ground station pronta (Windows): vídeo do robô, câmera da arena, telemetria, navegação, manipulador, botão Retransmitir
referencia/robo-pi/	Lado do robô no Raspberry Pi: robo_video.py (vídeo), robo_agente.py (MQTT: recebe comandos e publica telemetria), robo.sh + configurar_pi.sh (serviço no boot), medir.sh, encontrar_pi.ps1
referencia/gs-sem-tela/	Alternativas sem interface: gs_relay.py (repassa o vídeo por WHIP), sim_rover.py (robô simulado para testar)

> As referências são opcionais: a equipe pode fazer a sua GS e o seu software de robô. O contrato é obrigatório.

## Passo a passo

### 1. Credenciais
Pegue o cartão da sua equipe em `cartoes/roverNN.md`. O ID vale para MQTT e vídeo.
Prefixo para os tópicos de publicação: `rover/{id_equipe}/...`.

> Exemplos: `rover/4/telemetry`, `rover/4/tool_movement_commands`

### 2. Ground station 
No Windows, execute o PowerShell como administrador

```bash
cd C:\rover-gs                      
.\instalar.bat --servidor 192.168.3.2 --id roverNN --senha SUA_SENHA
```

A Ground Station é acessada em http://127.0.0.1:8070. O ⚙ mostra o IP desta GS, que é o que o robô usa para mandar o vídeo. 

> **Testar sem robô**: rode 
> `C:\RoverGS\simular_robo.bat` 
> ou 
> `C:\RoverGS\camera_robo.bat` (usa a webcam do PC)
> depois, **Retransmitir**. Confira em http://192.168.3.2:8080/arena.html?path=rover/roverNN

### 3. Robô
(Raspberry Pi, internet na primeira vez)

```bash
scp -r referencia\robo-pi usuario@IP_DO_PI:~/rover-pi
ssh usuario@IP_DO_PI
cd ~/rover-pi
nano robo.sh                        
sudo bash configurar_pi.sh          
journalctl -u rover -f              
```

### 4. Ligue o seu hardware
O robo_agente.py hoje só registra os comandos (LogDriver). Troque por uma classe com os mesmos quatro métodos:

```python
class MeuDriver:
    moving = False                             # 
    def drive(self, linear, angular): ...      # 
    def stop(self, motivo): ...                # 
    def arm(self, juntas, preset): ...         # 
    def gripper(self, acao): ...               # 
```

| Nome | Tipo | Default / Parâmetros | Descrição |
|---|---|---|---|
| `moving` | Atributo | False | Lido pelo homem-morto: True enquanto os motores giram |
| `drive()` | Método | `linear`, `angular` | ambos os parâmetros recebem valores entre -1 e 1; converter para PWM/velocidade; atualize self.moving |
| `stop()` | Método | `motivo` | para motores e define self.moving = False (chamado pelo homem-morto) |
| `arm()` | Método | `juntas`, `preset` | lista de ângulos em graus |
| `gripper()` | Método | `acao` | Recebe "open" e "close" |

O agente já cuida de três coisas:

1. homem-morto: para se ficar 0,5 s sem comando ou se perder o broker
2. confirmação do modo
3. status online/offline

No mesmo arquivo, troque a bateria simulada (telemetry_loop) pela leitura real e publique os estados reais. Ajuste o braço da GS (nomes, limites e posições prontas) em C:\RoverGS\config\gs.json → manipulador.

### 5. Publicação 
O que o dashboard e o juiz esperam (contrato §3.1)

`rover/<id_equipe>/state`: IDLE, BUSCA, NAVEGACAO, BUSCA_ARTEFATO, ARTEFATO_ENCONTRADO ou COLETA. O alerta de autonomia do juiz depende disso. Reporte com fidelidade.

rover/<id_equipe>/mode: MANUAL ou AUTONOMO. Pilotar em MANUAL durante NAVEGACAO ou BUSCA_ARTEFATO gera penalidade de tempo.

rover/<id>/battery: tensão, % e corrente reais. A referência simula esses valores.

rover/<id>/detection: uma mensagem por identificação YOLO (artifact ∈ hiperboloide, argola_pequena, argola_grande, cilindro, exotico).

6. Vídeo dentro do padrão (contrato §4.1): H.264 baseline, sem B-frames, 1 fatia, keyframe a cada 1 s, 640×480, ≤ 1500 kbps. O robo_video.py já gera assim.

7. Homologação com o juiz antes da prova: checklist no contrato §6.

Dicas do que já deu errado nos testes
Webcam USB barata só tem YUYV. O robo_video.py detecta sozinho; para forçar, use --formato yuyv422.
Imagem preta: outro programa está usando a câmera (app Câmera, Teams, OBS, Animaze).
Wi-Fi do Pi com taxa baixa: sudo iw dev wlan0 set power_save off. O configurar_pi.sh já deixa assim.
Pi na bateria: confira vcgencmd get_throttled. Se o último dígito não for 0, há subtensão agora. Alimente o Pi separado dos motores.
A porta 5000 da GS aceita uma origem de vídeo por vez: não rode o simulador junto com o robô.
scp de pasta precisa de -r.
