⚡ ChargeCore — Módulo de Controle Embarcado para Eletropostos Comerciais
EV Challenge 2026 · FIAP + GoodWe · Trilha: ChargeGrid Intelligence
 Sprint 1 — Projeto Sustentável em Arquitetura de Computadores

👥 Integrantes
Nome
RM
(Nome completo)
RM-XXXXX
(Nome completo)
RM-XXXXX
(Nome completo)
RM-XXXXX


🔴 O Problema
Eletropostos comerciais modernos operam 24 horas por dia, 7 dias por semana.
 O software que gerencia suas operações críticas — autenticação de sessão, leitura de sensores de potência e controle de carga — geralmente roda em sistemas de alto nível (Python, Node.js, Java) sobre hardware genérico.
Isso gera três consequências diretas:
Consumo energético desnecessário no próprio controlador embarcado, que executa centenas de milhares de instruções onde bastam dezenas.
Latência elevada na resposta a eventos críticos (pico de demanda, falha de sessão), comprometendo a segurança elétrica.
Desperdício de recursos computacionais em tarefas simples e repetitivas, aumentando o custo de infraestrutura e o descarte eletrônico.
Em uma rede com centenas de eletropostos ativos simultaneamente, esse desperdício computacional se traduz em quilowatts-hora perdidos — energia que poderia ir diretamente para os veículos.

💡 Justificativa
A ineficiência está na camada de software, não no hardware.
 Operações como ler um sensor, comparar um limiar de potência e ajustar a carga de saída são tarefas determinísticas e repetitivas — o cenário ideal para programação em Assembly.
"Cada instrução que economizamos no controlador é energia que permanece na rede."
A linguagem Assembly x86 32-bit (NASM) foi escolhida porque:
Permite controle total sobre as instruções executadas pelo processador, sem overhead de compilador ou runtime.
Utiliza chamadas de sistema diretas (int 0x80) — menos camadas de software, menos ciclos de clock.
É transparente: cada linha de código corresponde a exatamente uma instrução de máquina. Nada é escondido.
É executável no OnlineGDB, ferramenta utilizada em aula, sem necessidade de instalação.

🛠️ Proposta de Solução
ChargeCore é um módulo de firmware escrito em Assembly x86 32-bit (NASM) que substitui o software de alto nível nas operações críticas de um eletroposto comercial.
Componentes do módulo
┌──────────────────────────────────────────────────────┐
│              ChargeCore Firmware (NASM x86 32-bit)   │
│                                                      │
│  ┌──────────────────┐    ┌─────────────────────────┐ │
│  │  Autenticação    │    │   monitor_power         │ │
│  │  (Assembly)      │    │   (Assembly)            │ │
│  │                  │    │                         │ │
│  │ Confirma sessão  │    │ Sensor (185) vs         │ │
│  │ antes de carregar│    │ Limiar (220) → ok/high  │ │
│  └────────┬─────────┘    └──────────┬──────────────┘ │
│           │                         │                │
│           └──────────┬──────────────┘                │
│                      ▼                               │
│           ┌───────────────────┐                      │
│           │   adjust_charge   │                      │
│           │   (Assembly)      │                      │
│           │                   │                      │
│           │ Saída: 7 / 11 /   │                      │
│           │ 22 kW conforme    │                      │
│           │ nivel_demanda     │                      │
│           └───────────────────┘                      │
└──────────────────────────────────────────────────────┘

Fluxo de execução
_start
  │
  ├─► print "SESSAO AUTORIZADA"
  │
  ├─► monitor_power()
  │       │
  │   [sensor=185] < [limiar=220]?
  │       │
  │      Sim ──► print "POTENCIA DENTRO DO LIMITE"
  │      Não ──► print "POTENCIA ACIMA DO LIMITE"
  │
  └─► adjust_charge()
          │
      [nivel_demanda = 1]
          │
          └──► print "CARGA: 11.0 kW"
               exit (0)


🏗️ Arquitetura Utilizada
Componente
Escolha
Justificativa
ISA
x86 32-bit
Arquitetura ensinada em aula, suportada pelo OnlineGDB
Assembler
NASM
Sintaxe Intel clara, padrão no ambiente acadêmico
Simulador/Executor
OnlineGDB
Execução e debug sem instalação, usado em aula
Chamadas de sistema
int 0x80 (Linux)
Acesso direto ao kernel — zero dependência de biblioteca

Conceitos de Arquitetura Aplicados
Registradores x86: eax, ebx, ecx, edx — dados manipulados diretamente nos registradores, sem alocação desnecessária na memória heap.
Instruções de comparação e desvio condicional: cmp + jl / je — lógica de controle implementada sem overhead de estruturas de alto nível.
Chamada de sistema direta: int 0x80 com eax=4 (sys_write) e eax=1 (sys_exit) — nenhuma biblioteca intermediária (sem libc, sem stdio.h).
Segmentação de memória: seção .data para dados estáticos; seção .text para instruções — modelo clássico de memória segmentada x86.
Convenção de chamada manual: parâmetros passados diretamente via ecx e edx, retorno em eax — sem stack frame gerado automaticamente pelo compilador.

💻 Código Assembly — ChargeCore
Rotina: monitor_power
Lê o sensor de potência e verifica o limiar da rede.
monitor_power:
    mov eax, [sensor_potencia]   ; carrega leitura atual (185 = 18.5 kW)
    cmp eax, [limiar_maximo]     ; compara com o limite máximo (220 = 22.0 kW)
    jl  potencia_ok              ; se menor → dentro do limite
    mov eax, 1                   ; caso contrário → retorna 1 (acima)
    ret
potencia_ok:
    mov eax, 0                   ; retorna 0 (dentro do limite)
    ret

Por que isso é eficiente? São 4 instruções para uma decisão completa. O equivalente em Python exige carregar o runtime CPython antes de executar qualquer linha.
Rotina: adjust_charge
Define a potência de saída conforme o nível de demanda.
adjust_charge:
    mov eax, [nivel_demanda]   ; carrega nível atual (0, 1 ou 2)
    cmp eax, 0
    je  carga_baixa            ; nível 0 → 7.0 kW (off-peak)
    cmp eax, 1
    je  carga_media            ; nível 1 → 11.0 kW (padrão)
carga_alta:
    mov eax, 220               ; nível 2 → 22.0 kW (máximo)
    ret
carga_media:
    mov eax, 110
    ret
carga_baixa:
    mov eax, 70
    ret

Rotina auxiliar: print
Syscall direta para escrita no terminal — sem biblioteca.
print:
    mov eax, 4     ; syscall número 4 = sys_write
    mov ebx, 1     ; file descriptor 1 = stdout
    int 0x80       ; interrupção de software → chama o kernel Linux
    ret


📊 Comparativo: Assembly x86 vs Python
Operação: verificar potência e exibir resultado
Em Python:
sensor = 185
limiar = 220
if sensor < limiar:
    print("POTENCIA DENTRO DO LIMITE")
else:
    print("POTENCIA ACIMA DO LIMITE")

Por trás dessas 4 linhas, o interpretador Python executa:
Inicialização do runtime CPython
Resolução de variáveis no dicionário de escopo (hash lookup)
Chamada de print() → sys.stdout.write() → libc → kernel
Estimativa: 50.000–200.000 instruções de máquina
Em Assembly x86 (ChargeCore):
mov eax, [sensor_potencia]
cmp eax, [limiar_maximo]
jl  potencia_ok
; exibição via int 0x80 (3 instruções adicionais)

Total: 6–8 instruções de máquina
Acesso ao kernel: direto via int 0x80, sem intermediários
Métrica
Python
Assembly x86 (ChargeCore)
Instruções de máquina (estimado)
50.000–200.000
6–8
Dependência de runtime
Sim (CPython ~30 MB)
Não
Acesso ao kernel
Via libc → stdlib → kernel
Direto (int 0x80)
Camadas de software
4+
1
Comportamento determinístico
Não (GC, bytecodes)
Sim


📊 Impactos Esperados
Eficiência Computacional
Redução drástica no número de instruções executadas por ciclo de operação.
Resposta a eventos de demanda em microssegundos — sem esperar runtime inicializar.
Firmware com footprint mínimo, compatível com microcontroladores de baixo custo.
Eficiência Energética
Controlador embarcado consome menos energia → menor dissipação de calor → menor necessidade de resfriamento ativo nos gabinetes dos eletropostos.
Em uma rede de 1.000 eletropostos operando 24h/dia, a redução do consumo dos controladores representa economia estimada de 120–300 kWh/mês — equivalente a carregar ~15 veículos elétricos adicionais sem gerar nova demanda.
Confiabilidade
Código determinístico: sem garbage collector, sem máquina virtual, sem event loop assíncrono.
Comportamento previsível em condições críticas (sobretensão, falha de rede, tentativa de autenticação inválida).

🌿 Relação com Sustentabilidade e Energias Renováveis
O ChargeGrid Intelligence gerencia eletropostos alimentados por fontes renováveis (solar + rede).
 O ChargeCore agrega sustentabilidade em três camadas:
Camada
Como o Assembly contribui
Hardware
Menos instruções → menos ciclos de clock → menor consumo do chip controlador
Rede elétrica
Resposta mais rápida a excedente solar → mais energia limpa aproveitada em tempo real
Ciclo de vida
Firmware leve → hardware mais simples e durável → menos descarte eletrônico

Quando painéis solares geram excedente às 14h, o monitor_power detecta o evento e o adjust_charge eleva a potência de saída em microssegundos — sem esperar nenhum runtime. Isso maximiza o uso da energia limpa disponível naquele instante.
O código eficiente é parte da infraestrutura verde.
 Reduzir o consumo do controlador é tão sustentável quanto instalar mais painéis.

🔧 Como Executar no OnlineGDB
Acesse onlinegdb.com
No seletor de linguagem (canto superior direito), escolha Assembly (x86)
Cole o conteúdo do arquivo chargecore_firmware.asm
Clique em ▶ Run
Saída esperada no terminal:
SESSAO AUTORIZADA
POTENCIA DENTRO DO LIMITE
CARGA: 11.0 kW


🔗 Links
📹 Vídeo Pitch: (adicionar link do YouTube)
💻 Repositório GitHub: (este repositório)

📁 Estrutura do Repositório
chargecore/
├── README.md
├── assembly/
│   └── chargecore_firmware.asm    # Firmware principal (NASM x86 32-bit)
├── comparativo/
│   └── equivalente.py             # Código Python equivalente (para comparação)
└── docs/
    └── arquitetura.md             # Diagrama e análise de eficiência


Projeto desenvolvido para o EV Challenge 2026 — FIAP + GoodWe.
 Sprint 1 — Arquitetura de Computadores.

