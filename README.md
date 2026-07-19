# PrimeFeeder - Alimentador e Posicionador Automático

Este projeto consiste em um firmware definitivo para o controle de um sistema automatizado baseado em ESP32-C3, composto por dois motores (um contínuo para movimentação de carga/balde e um posicional para alimentação/pescador), sensores ópticos de segurança e monitoramento de fluxo, além de proteção nativa contra travamentos mecânicos e falta de insumos (espoletas).

## 📋 Sumário
- [Hardware Utilizado](#-hardware-utilizado)
- [Mapeamento de Pinos (Pinout)](#-mapeamento-de-pinos-pinout)
- [Esquema de Ligação Elétrica](#-esquema-de-liga%C3%A7%C3%A3o-el%C3%A9trica)
- [Lógicas de Proteção do Sistema](#-l%C3%B3gicas-de-prote%C3%A7%C3%A3o-do-sistema)
- [Configuração Inicial e OTA](#%EF%B8%8F-configura%C3%A7%C3%A3o-inicial-e-ota)

---

## 🛠️ Hardware Utilizado

1. **Microcontrolador:** ESP32-C3 (Ex: Xiao ESP32-C3 ou NodeMCU ESP32-C3).
2. **Motor 1 (Balde):** Servo Motor de Rotação Contínua (360 graus) - Ex: MG995/MG996R modificados ou FS5103R.
3. **Motor 2 (Pescador):** Servo Motor Posicional Padrão (180 graus) - Ex: MG90S (metálico) ou SG90.
4. **Sensor 1 (Fluxo de Espoletas):** Sensor Óptico de Slot Omron **EE-SX672-WR** (Saída NPN).
5. **Sensor 2 (Fim de Curso/Obstrução M2):** Sensor óptico ou Microswitch (Lógica HIGH quando bloqueado).
6. **Alarme Sonoro:** Buzzer Piezoelétrico Ativo (5V).
7. **Fonte de Alimentação:** Fonte chaveada de 5V DC com capacidade mínima de 2A a 3A (os servos demandam pico de corrente ao iniciarem o movimento).

---

## 📌 Mapeamento de Pinos (Pinout)

No ESP32-C3, os pinos foram distribuídos estrategicamente para evitar conflitos com os pinos de boot (*strapping pins*), garantindo que a placa inicialize corretamente em qualquer cenário.

| Componente | Tipo | Pino ESP32-C3 | Detalhes Técnicos |
| :--- | :--- | :--- | :--- |
| **Servo Motor 1 (Balde)** | Saída PWM | `GPIO 3` | Sinal de controle do servo contínuo |
| **Servo Motor 2 (Pescador)** | Saída PWM | `GPIO 2` | Sinal de controle do servo posicional |
| **Sensor Omron EE-SX672** | Entrada Digital | `GPIO 4` | Configurado com `INPUT_PULLUP` interno |
| **Sensor Obstáculo M2** | Entrada Digital | `GPIO 10` | Configurado como `INPUT` padrão |
| **Buzzer Alarme** | Saída Digital | `GPIO 8` | Controle de bipes de alerta |

---

## 🔌 Esquema de Ligação Elétrica

### 1. Sensor Óptico Omron EE-SX672-WR (Sensor 1)
O sensor Omron opera em lógica NPN (coletor aberto). Para casar perfeitamente com a lógica de 3.3V do ESP32-C3 sem o uso de resistores externos, o sensor é alimentado em 5V e o pino do microcontrolador usa o pull-up interno.
* **Fio Marrom:** Conectar ao pino **5V** (ou VIN) do ESP32.
* **Fio Azul:** Conectar ao pino **GND** do ESP32.
* **Fio Preto (Sinal):** Conectar diretamente ao pino **GPIO 4** do ESP32.
* **Fio Branco (Modo):** Deixar **desconectado e isolado** (Garante o modo *Dark-ON*, onde a saída satura em `LOW` ao cortar o feixe de luz).

### 2. Servo Motores (M1 e M2)
* **Fio Vermelho (VCC):** Conectar à linha positiva de **5V** da fonte.
* **Fio Marrom/Preto (GND):** Conectar ao **GND** comum do circuito (Fonte + ESP32).
* **Fio Amarelo/Laranja (Sinal):** 
  * Servo 1 no pino **GPIO 3**.
  * Servo 2 no pino **GPIO 2**.

*Nota: Nunca alimente o VCC dos servos diretamente pelos pinos de saída regulada de 3.3V do ESP32, pois isso causará resets constantes por subtensão devido à alta demanda de corrente.*

### 3. Buzzer
* **Pino (+):** Conectar ao pino **GPIO 8** do ESP32.
* **Pino (-):** Conectar ao **GND**.

---

## 🔒 Lógicas de Proteção do Sistema

O firmware conta com mecanismos de segurança industriais para proteger os motores e evitar desperdício de tempo e insumos:

### A. Proteção por Falta de Espoletas (Motor 1)
O Sensor 1 monitora a passagem das espoletas. Caso o motor 1 esteja ligado e nenhuma espoleta passe pelo sensor dentro do tempo configurado na interface web, o sistema entra em regime de proteção em camadas:
1. **Alarme 1:** Dispara o buzzer por 3 segundos e para o motor.
2. **Pausa de Segurança:** Mantém o sistema parado por 15 segundos.
3. **Segunda Tentativa:** Liga o motor novamente para tentar puxar novos insumos. Se falhar outra vez, entra em **BLOQUEADO**. O motor só volta a operar após o operador clicar no botão **REARMAR MOTOR SISTEMA** na interface Web.

### B. Sistema Anti-Trava Inteligente (Motor 2)
Projetado para identificar obstruções completas ou restrições parciais de curso no pescador (Motor 2). O firmware calcula dinamicamente o tempo máximo teórico que o curso leva para ir do ângulo Mínimo ao Máximo:

$$\text{Tempo Limite} = (\text{Ângulo Máximo} - \text{Ângulo Mínimo}) \times \text{Suavidade (ms)} + 10\text{ segundos de margem}$$

Se o Motor 2 estiver ligado e não conseguir alternar sua direção de curso dentro desse limite de tempo, o firmware desliga o motor imediatamente por segurança e dispara um padrão de **5 apitos curtos com intervalo de 1 segundo** para alertar o operador sobre a falha mecânica.

---

## 🌐 Configuração Inicial e OTA

### Primeiro Acesso (Access Point)
No primeiro boot, ou caso o ESP32 não encontre a rede Wi-Fi configurada, ele abrirá um Ponto de Acesso próprio:
* **SSID:** `PrimeFeeder`
* **Senha:** `12345678`
* **IP de Configuração:** `192.168.10.200` ou através do endereço [http://primefeeder.local](http://primefeeder.local) (via mDNS).

Na interface gerada, selecione a sua rede sem fio na lista de escaneamento, digite a senha e clique em **Salvar e Conectar**. O dispositivo irá reiniciar e assumir um IP na sua rede local.

### Atualização via Nuvem (Firmware Over-The-Air)
O ecossistema está preparado para atualizações com um clique integradas ao repositório do GitHub. O painel no rodapé da página busca o arquivo descritor de versão (`versao.txt`) aplicando parâmetros anti-cache. 
Se uma versão mais recente for detectada na nuvem, um popup solicitará a autorização do usuário e efetuará o download seguro (`WiFiClientSecure`) e a gravação na memória flash de maneira totalmente automatizada. Caso o servidor esteja inacessível, a seção disponibiliza um formulário para **Upload Manual** do arquivo `.bin` compilado.
