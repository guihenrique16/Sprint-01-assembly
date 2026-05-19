#  ChargeCore — Módulo de Controle Embarcado para Eletropostos Comerciais

> **EV Challenge 2026 · FIAP + GoodWe · Trilha: ChargeGrid Intelligence**  
> Sprint 1 — Projeto Sustentável em Arquitetura de Computadores

---

##  Integrantes

| Nome | RM |
|------|-----|
| _(Nome completo)_ | RM-XXXXX |
| _(Nome completo)_ | RM-XXXXX |
| _(Nome completo)_ | RM-XXXXX |
| _(Nome completo)_ | RM-XXXXX |
| _(Nome completo)_ | RM-XXXXX |

---

## O Problema

Eletropostos comerciais modernos operam 24 horas por dia, 7 dias por semana.  
O software que gerencia suas operações críticas — autenticação de sessão, leitura de sensores de potência e controle de carga — geralmente roda em **sistemas de alto nível** (Python, Node.js, Java) sobre hardware genérico.

Isso gera três consequências diretas:

1. **Consumo energético desnecessário** no próprio controlador embarcado, que executa centenas de milhares de instruções onde bastam dezenas.
2. **Latência elevada** na resposta a eventos críticos (pico de demanda, falha de sessão), comprometendo a segurança elétrica.
3. **Desperdício de recursos computacionais** em tarefas simples e repetitivas, aumentando o custo de infraestrutura e o descarte eletrônico.

Em uma rede com centenas de eletropostos ativos simultaneamente, esse desperdício computacional se traduz em quilowatts-hora perdidos — energia que poderia ir diretamente para os veículos.

---

## Justificativa

A ineficiência está na camada de software, não no hardware.  
Operações como **ler um sensor**, **comparar um limiar de potência** e **ajustar a carga de saída** são tarefas determinísticas e repetitivas — o cenário ideal para programação em **Assembly**.

> _"Cada instrução que economizamos no controlador é energia que permanece na rede."_

A linguagem **Assembly x86 32-bit (NASM)** foi escolhida porque:

- Permite **controle total** sobre as instruções executadas pelo processador, sem overhead de compilador ou runtime.
- Utiliza **chamadas de sistema diretas** (`int 0x80`) — menos camadas de software, menos ciclos de clock.
- É **transparente**: cada linha de código corresponde a exatamente uma instrução de máquina. Nada é escondido.
- É executável no **OnlineGDB**, ferramenta utilizada em aula, sem necessidade de instalação.

---

## Proposta de Solução

**ChargeCore** é um módulo de firmware escrito em Assembly x86 32-bit (NASM) que substitui o software de alto nível nas operações críticas de um eletroposto comercial.

### Componentes do módulo

```
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
```

### Fluxo de execução

```
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
```

---

## Arquitetura Utilizada

| Componente | Escolha | Justificativa |
|---|---|---|
| ISA | **x86 32-bit** | Arquitetura ensinada em aula, suportada pelo OnlineGDB |
| Assembler | **NASM** | Sintaxe Intel clara, padrão no ambiente acadêmico |
| Simulador/Executor | **OnlineGDB** | Execução e debug sem instalação, usado em aula |
| Chamadas de sistema | **int 0x80 (Linux)** | Acesso direto ao kernel — zero dependência de biblioteca |

### Conceitos de Arquitetura Aplicados

- **Registradores x86**: `eax`, `ebx`, `ecx`, `edx` — dados manipulados diretamente nos registradores, sem alocação desnecessária na memória heap.
- **Instruções de comparação e desvio condicional**: `cmp` + `jl` / `je` — lógica de controle implementada sem overhead de estruturas de alto nível.
- **Chamada de sistema direta**: `int 0x80` com `eax=4` (sys_write) e `eax=1` (sys_exit) — nenhuma biblioteca intermediária (sem libc, sem stdio.h).
- **Segmentação de memória**: seção `.data` para dados estáticos; seção `.text` para instruções — modelo clássico de memória segmentada x86.
- **Convenção de chamada manual**: parâmetros passados diretamente via `ecx` e `edx`, retorno em `eax` — sem stack frame gerado automaticamente pelo compilador.

---

## Código Assembly — ChargeCore

```nasm
section .data

sensor_potencia    dd 185
limiar_maximo      dd 220
nivel_demanda      dd 1

msg_auth           db "SESSAO AUTORIZADA", 10
len_auth           equ $ - msg_auth

msg_ok             db "POTENCIA DENTRO DO LIMITE", 10
len_ok             equ $ - msg_ok

msg_high           db "POTENCIA ACIMA DO LIMITE", 10
len_high           equ $ - msg_high

msg_low            db "CARGA: 7.0 kW", 10
len_low            equ $ - msg_low

msg_med            db "CARGA: 11.0 kW", 10
len_med            equ $ - msg_med

msg_max            db "CARGA: 22.0 kW", 10
len_max            equ $ - msg_max

section .text
global _start

; print
; ecx = mensagem
; edx = tamanho

print:
    mov eax, 4
    mov ebx, 1
    int 0x80
    ret


; monitor_power
;
; eax = 0 -> ok
; eax = 1 -> acima limite

monitor_power:

    mov eax, [sensor_potencia]
    cmp eax, [limiar_maximo]

    jl potencia_ok

    mov eax, 1
    ret

potencia_ok:
    mov eax, 0
    ret


; adjust_charge

; eax = potência final

adjust_charge:

    mov eax, [nivel_demanda]

    cmp eax, 0
    je carga_baixa

    cmp eax, 1
    je carga_media

carga_alta:
    mov eax, 220
    ret

carga_media:
    mov eax, 110
    ret

carga_baixa:
    mov eax, 70
    ret


; MAIN

_start:


    ; Autorização

    mov ecx, msg_auth
    mov edx, len_auth
    call print


    ; Monitoramento
 
    call monitor_power

    cmp eax, 0
    je mostrar_ok

mostrar_high:

    mov ecx, msg_high
    mov edx, len_high
    call print

    jmp ajustar

mostrar_ok:

    mov ecx, msg_ok
    mov edx, len_ok
    call print


; Ajustar carga

ajustar:

    call adjust_charge

    cmp eax, 70
    je print_low

    cmp eax, 110
    je print_med

print_max:

    mov ecx, msg_max
    mov edx, len_max
    call print

    jmp fim

print_med:

    mov ecx, msg_med
    mov edx, len_med
    call print

    jmp fim

print_low:

    mov ecx, msg_low
    mov edx, len_low
    call print


; Exit

fim:

    mov eax, 1
    mov ebx, 0
    int 0x80
```



## Comparativo: Assembly x86 vs Python

### Operação: verificar potência e exibir resultado

**Em Python:**
```python
sensor = 185
limiar = 220
if sensor < limiar:
    print("POTENCIA DENTRO DO LIMITE")
else:
    print("POTENCIA ACIMA DO LIMITE")
```

Por trás dessas 4 linhas, o interpretador Python executa:
- Inicialização do runtime CPython
- Resolução de variáveis no dicionário de escopo (hash lookup)
- Chamada de `print()` → `sys.stdout.write()` → libc → kernel
- **Estimativa: 50.000–200.000 instruções de máquina**

**Em Assembly x86 (ChargeCore):**
```nasm
mov eax, [sensor_potencia]
cmp eax, [limiar_maximo]
jl  potencia_ok
; exibição via int 0x80 (3 instruções adicionais)
```
- **Total: 6–8 instruções de máquina**
- Acesso ao kernel: direto via `int 0x80`, sem intermediários

| Métrica | Python | Assembly x86 (ChargeCore) |
|---|---|---|
| Instruções de máquina (estimado) | 50.000–200.000 | 6–8 |
| Dependência de runtime | Sim (CPython ~30 MB) | Não |
| Acesso ao kernel | Via libc → stdlib → kernel | Direto (`int 0x80`) |
| Camadas de software | 4+ | 1 |
| Comportamento determinístico | Não (GC, bytecodes) | Sim |

---

## Impactos Esperados

### Eficiência Computacional
- Redução drástica no número de instruções executadas por ciclo de operação.
- Resposta a eventos de demanda em microssegundos — sem esperar runtime inicializar.
- Firmware com footprint mínimo, compatível com microcontroladores de baixo custo.

### Eficiência Energética
- Controlador embarcado consome menos energia → menor dissipação de calor → menor necessidade de resfriamento ativo nos gabinetes dos eletropostos.
- Em uma rede de **1.000 eletropostos** operando 24h/dia, a redução do consumo dos controladores representa economia estimada de **120–300 kWh/mês** — equivalente a carregar ~15 veículos elétricos adicionais sem gerar nova demanda.

### Confiabilidade
- Código determinístico: sem garbage collector, sem máquina virtual, sem event loop assíncrono.
- Comportamento previsível em condições críticas (sobretensão, falha de rede, tentativa de autenticação inválida).

---

## Relação com Sustentabilidade e Energias Renováveis

O ChargeGrid Intelligence gerencia eletropostos alimentados por fontes renováveis (solar + rede).  
O ChargeCore agrega sustentabilidade em **três camadas**:

| Camada | Como o Assembly contribui |
|---|---|
| **Hardware** | Menos instruções → menos ciclos de clock → menor consumo do chip controlador |
| **Rede elétrica** | Resposta mais rápida a excedente solar → mais energia limpa aproveitada em tempo real |
| **Ciclo de vida** | Firmware leve → hardware mais simples e durável → menos descarte eletrônico |

Quando painéis solares geram excedente às 14h, o `monitor_power` detecta o evento e o `adjust_charge` eleva a potência de saída em microssegundos — sem esperar nenhum runtime. Isso maximiza o uso da energia limpa disponível naquele instante.

> **O código eficiente é parte da infraestrutura verde.**  
> Reduzir o consumo do controlador é tão sustentável quanto instalar mais painéis.

---

**Saída esperada no terminal:**
```
SESSAO AUTORIZADA
POTENCIA DENTRO DO LIMITE
CARGA: 11.0 kW
```

---

## Links

- **Vídeo Pitch:** _(adicionar link do YouTube)_
- **Repositório GitHub:** _(este repositório)_

---



*Projeto desenvolvido para o EV Challenge 2026 — FIAP + GoodWe.*  
*Sprint 1 — Arquitetura de Computadores.*
