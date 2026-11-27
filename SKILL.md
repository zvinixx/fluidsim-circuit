---
name: fluidsim-circuit
description: "Cria e edita projetos de circuito hidráulico e eletrohidráulico do FluidSIM 4 (Festo) gerando o arquivo .ct pronto para abrir com duplo clique, sem precisar abrir o programa. Use quando o pedido mencionar 'circuito FluidSIM', 'arquivo .ct', 'projeto de hidráulica', 'montar um pistão', 'cilindro com válvula direcional', 'fim de curso', 'sensor de posição no cilindro', 'circuito eletrohidráulico com solenoide', 'GRAFCET', 'editar um .ct' ou 'renomear componentes do circuito'."
license: MIT
metadata:
  author: zvinixx
  version: "1.0.0"
  fluid_sim: "4.x Hidráulica"
---

# Skill: circuitos FluidSIM `.ct`

Monta projetos hidráulicos e eletrohidráulicos do **FluidSIM 4** gerando o
arquivo `.ct` final — sem abrir o programa.

**Autor original da engenharia reversa e da skill: [zvinixx](https://github.com/zvinixx).**

---

## Regra número um (leia antes de tudo)

O `.ct` é um formato **binário comprimido**, e a compressão não pode ser
reescrita do zero. Por isso esta skill trabalha **a partir de circuitos-base
reais**: ela escolhe o circuito mais próximo do pedido, copia e ajusta.

| Você quer… | Dá? |
|---|---|
| Circuito com pistão, válvula, bomba, filtro | ✅ |
| Pistão com fim de curso | ✅ (bases prontas) |
| Motor hidráulico | ✅ |
| Escada elétrica, solenoide, relé | ✅ |
| GRAFCET / sequência | ✅ |
| Renomear componentes | ✅ |
| Mudar valores físicos (curso, massa) | ⚠️ funciona, validar depois |
| Desenhar circuito novo do zero | ❌ **não é possível** |

Se o pedido não existir no catálogo, **diga o que falta**. Nunca entregue um
arquivo gerado do zero: o FluidSIM rejeita com `Unknown file format`.

---

## Passo 1 — Traduza o pedido em componentes

| O usuário fala… | Componente |
|---|---|
| pistão, cilindro que abre e fecha, atuador | `cilindro` |
| válvula direcional, comando, 4/2, 4/3, 5/2 | `valvula_dir` |
| bomba, unidade hidráulica, fonte de pressão | `bomba` |
| tanque, reservatório | `reservatorio` |
| filtro | `filtro` |
| válvula de retenção, anti-retorno | `valvula_ret` |
| válvula de pressão, alívio, redutora | `valvula_press` |
| estrangulamento, controle de vazão | `estrangul` |
| acumulador | `acumulador` |
| manômetro | `manometro` |
| pressostato | `pressostato` |
| motor hidráulico | `motor` |
| **fim de curso, sensor indutivo, fim de posição, sensor de proximidade** | `fim_de_curso` |
| solenoide, eletroválvula, bobina | `solenoide` |
| botoeira, chave, botão | `botoeira` |
| relé, contator | `rele` |
| lâmpada, sinaleiro | `lampada` |
| sequência, passo a passo, GRAFCET | `grafcet` |

---

## Passo 2 — Escolha o circuito-base

Consulte a tabela completa no §7. Bases mais usadas:

| Base | Componentes | Observação |
|---|---|---|
| `ct/demo_02.ct` | cilindro, valvula_dir, reservatorio | pistão básico com válvula manual |
| `ct/Shows/dida_042.ct` | cilindro, valvula_dir, bomba, filtro | pistão + bomba + filtro |
| `ct/Shows/dida_037.ct` | cilindro, valvula_dir, bomba, reservatorio | pistão + unidade completa |
| `ct/tp601/TP601_11b.ct` | cilindro, valvula_dir, fim_de_curso, solenoide, bomba | **1 cilindro + 2 fins de curso + escada** |
| `ct/Shows/dida_173d.ct` | cilindro, valvula_dir, fim_de_curso, bomba, valvula_ret, valvula_press | **2 cilindros + fim de curso + pressão** |
| `ct/tp501/TP501_20.ct` | cilindro, valvula_dir, fim_de_curso, bomba, valvula_ret | cilindro + fim de curso + bomba |
| `ct/tp701/TP701_09.ct` | cilindro, valvula_dir, fim_de_curso, filtro | cilindro + fim de curso + filtro |
| `ct/tp602/TP602_01.ct` | cilindro, valvula_dir, fim_de_curso, reservatorio, lampada | cilindro + fim de curso + lâmpada |
| `ct/tp602/TP602_06.ct` | cilindro, … fim_de_curso, bomba, valvula_press, lampada | completo + sinaleiro |
| `ct/tp601/TP601_08b.ct` | cilindro, valvula_dir, solenoide, bomba | **eletrohidráulico com escada e tabela de estados** |
| `ct/tp601/TP601_01b.ct` | cilindro, valvula_dir, solenoide, botoeira | solenoide + botoeira |
| `ct/Shows/dida_070.ct` | motor, valvula_dir, valvula_ret | motor hidráulico |
| `ct/tp502/TP502_01.ct` | motor, valvula_dir, bomba | motor + bomba |
| `ct/Grafcet18.ct` | cilindro, valvula_dir, fim_de_curso, grafcet | **GRAFCET + cilindro + fim de curso** |
| `ct/tp502/TP502_05a.ct` | cilindro, valvula_dir, estrangul | controle de velocidade |
| `ct/tp502/TP502_10.ct` | cilindro, valvula_dir, bomba, reservatorio | dois cilindros + diagrama |

---

## Passo 3 — Gere o arquivo

O arquivo `.ct` é montado **copiando a base** e renomeando rótulos quando
necessário. Implementação de referência (Python 3, sem dependências):

```python
# build.py  —  gera um .ct a partir de uma base
import shutil, sys

BASE = r'D:\Hydraulic\ct\Shows\dida_042.ct'   # circuito escolhido
DEST = 'Projeto.ct'

shutil.copyfile(BASE, DEST)
print('gerado:', DEST)
```

A cópia já é um `.ct` válido — **duplo clique abre**.

### Renomear um componente

Trocas de texto seguem a regra do **mesmo comprimento** (complete com espaços):

```python
import ctcodec  # ver §6

n = ctcodec.patch_text(
    'Projeto.ct', 'Projeto.ct',
    [('Zylinder, doppeltwirkend', 'Cilindro principal  ')])
print('substituicoes:', n)
```

> Texto novo maior/menor desloca os índices internos e o desenho corrompe —
> embora o arquivo ainda abra.

---

## Passo 4 — Valide antes de entregar

```python
raw = bytes(b ^ 0x9A for b in open('Projeto.ct', 'rb').read())
toks = ctcodec.stream_info(raw)          # ver §6
usado = sum(t['size'] for t in toks)
assert usado == len(raw), 'fluxo nao cobre o arquivo'
```

Se der erro, tente a **segunda opção** da tabela do §7.

Validação ideal: abrir no FluidSIM e conferir o título da janela. Se a janela
mostrar `FluidSIM` (sem o nome do arquivo), houve `Unknown file format`.

---
