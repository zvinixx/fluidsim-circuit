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

## Passo 5 — Como colocar um pistão (explicação para o usuário)

No FluidSIM, manualmente:

1. **Biblioteca → Hidráulica → Atuadores** → arraste "Cilindro de dupla ação".
2. Botão direito no cilindro → **Propriedades**:

   | Campo | Significado |
   |---|---|
   | `x_pos` / `y_pos` | posição na folha |
   | `stroke` | curso em mm (padrão 200) |
   | `length` | comprimento desenhado |
   | `MASS` | massa da carga (kg) |
   | `DAMPING_ADJ_L` | amortecimento |

3. **Biblioteca → Válvulas → Válvula direcional** → conecte A/B ao cilindro e
   P/T à bomba e ao tanque.

No arquivo `.ct`, cada item vira propriedade textual:

```
 F x_pos I 120
 F y_pos I 240
 F stroke I 200
 F MASS I 9
 S sig_color I 65535
```

---

## Passo 6 — Como colocar um fim de curso

### No FluidSIM (manual)

1. Abra o circuito que já tem o cilindro.
2. **Biblioteca → Elétrica → Chaves → Sensor de proximidade**
   (*Näherungsschalter*).
3. Arraste o sensor **até a haste** do cilindro — ele encaixa e passa a
   detectar a posição.
4. Ligue o contato **13/14** na bobina do relé ou na entrada da lógica.
5. Para o segundo fim de curso, repita na outra extremidade.

O FluidSIM posiciona o sensor em `pos_l_` (esquerda) ou `pos_r_` (direita)
conforme onde você soltar na haste.

### No arquivo `.ct`

O sensor aparece como símbolo da biblioteca elétrica
(`sym/electr/switch/Proximit/px1..px4.ct`) e gera estas propriedades:

```
 S ProximitySwitch B F
 F pos_l_<n> I <coordenada>
 F pos_r_<n> I <coordenada>
```

Na escada elétrica aparece como contato `IS1` / `IS2` nos bornes 13/14.

### Adicionar a um circuito que não tem

**Não é suportado nesta versão.** Escolha uma base que já possua a quantidade
desejada — ver as bases com `fim_de_curso` no §7.

---

## Passo 7 — Catálogo de bases

Caminhos relativos a `D:\Hydraulic\`. Todas as 201 bases foram validadas
abrindo no FluidSIM.

| Base | Componentes | Observação |
|---|---|---|
| `ct/Grafcet/Grafcet05.ct` | grafcet | grafcet básico |
| `ct/Grafcet18.ct` | cilindro, valvula_dir, fim_de_curso, grafcet | verificado — GRAFCET + cilindro + fim de curso |
| `ct/Shows/dida_037.ct` | cilindro, valvula_dir, bomba, reservatorio | pistão + unidade completa |
| `ct/Shows/dida_042.ct` | cilindro, valvula_dir, bomba, filtro | pistão + bomba + filtro |
| `ct/Shows/dida_070.ct` | motor, valvula_dir, valvula_ret | motor + retenção |
| `ct/Shows/dida_173d.ct` | cilindro, valvula_dir, fim_de_curso, bomba, valvula_ret, valvula_press | verificado — 2 cilindros + fim de curso + pressão |
| `ct/demo_02.ct` | cilindro, valvula_dir, reservatorio | pistão básico |
| `ct/demo_03.ct` | cilindro, valvula_dir, bomba, reservatorio | pistão + bomba + válvula 4/3 |
| `ct/elect_02.ct` | cilindro, valvula_dir, solenoide, reservatorio | elétrico básico |
| `ct/tp501/TP501_20.ct` | cilindro, valvula_dir, fim_de_curso, bomba, valvula_ret | cilindro + fim de curso + bomba |
| `ct/tp502/TP502_01.ct` | motor, valvula_dir, bomba | motor + válvula |
| `ct/tp502/TP502_05a.ct` | cilindro, valvula_dir, estrangul | controle de velocidade |
| `ct/tp502/TP502_10.ct` | cilindro, valvula_dir, bomba, reservatorio | dois cilindros + diagrama |
| `ct/tp601/TP601_01b.ct` | cilindro, valvula_dir, solenoide, botoeira | solenoide + botoeira |
| `ct/tp601/TP601_08b.ct` | cilindro, valvula_dir, solenoide, bomba | verificado — eletrohidráulico completo |
| `ct/tp601/TP601_11b.ct` | cilindro, valvula_dir, fim_de_curso, solenoide, bomba | verificado — 1 cilindro + 2 fins de curso + escada |
| `ct/tp602/TP602_01.ct` | cilindro, valvula_dir, fim_de_curso, reservatorio, lampada | cilindro + fim de curso + lâmpada |
| `ct/tp602/TP602_06.ct` | cilindro, valvula_dir, fim_de_curso, bomba, reservatorio, valvula_press, lampada | completo + sinaleiro |
| `ct/tp701/TP701_09.ct` | cilindro, valvula_dir, fim_de_curso, filtro | cilindro + fim de curso + filtro |

Existem 182 bases adicionais em `D:\Hydraulic\ct\`. Para descobrir o que cada
uma tem, rode o script do §6 (`describe`).

---

## §6 — Implementação de referência do codec

Copie este arquivo como `ctcodec.py` ao lado do seu script. Sem dependências.

```python
#!/usr/bin/env python3
"""Codec dos arquivos .ct do FluidSIM 4.

Formato:
  * todo o arquivo recebe XOR 0x9A
  * depois disso e um fluxo LZ77:
      byte >= 0x80 -> literal: comprimento = b - 0x80, seguido dos bytes
      byte <  0x80 -> codigo de 16 bits (big-endian):
            comprimento = (code >> SHIFT) + 1
            offset      = code & (W - 1)
            comprimento == 1 -> RLE (valor no byte seguinte; repeticoes = offset)
            comprimento == 2 -> COMPRIMENTO ESTENDIDO (byte seguinte = tamanho)

  Modo B (padrao): W = 2048, SHIFT = 11
  Modo A:          W = 512,  SHIFT = 9

Nao existe cabecalho: o fluxo comeca no byte 0.
"""
KEY = 0x9A


def xor(data, key=KEY):
    return bytes(b ^ key for b in data)


def read(path):
    with open(path, 'rb') as f:
        return f.read()


def write(path, data):
    with open(path, 'wb') as f:
        f.write(data)


def stream_info(stream, shift=11, ring=2048):
    """Lista de tokens com posicao e tamanho."""
    mask = ring - 1
    toks = []
    i = 0
    n = len(stream)
    while i < n:
        b = stream[i]
        if b & 0x80:
            cnt = b - 0x80
            if i + 1 + cnt > n:
                break
            toks.append({'kind': 'lit', 'pos': i, 'size': 1 + cnt,
                         'data': stream[i + 1:i + 1 + cnt]})
            i += 1 + cnt
            continue
        if i + 2 > n:
            break
        code = (stream[i] << 8) | stream[i + 1]
        length = (code >> shift) + 1
        offset = code & mask
        if length == 1:
            if i + 3 > n:
                break
            toks.append({'kind': 'rle', 'pos': i, 'size': 3,
                         'value': stream[i + 2], 'count': offset})
            i += 3
            continue
        if length == 2:
            if i + 3 > n:
                break
            toks.append({'kind': 'copy', 'pos': i, 'size': 3,
                         'offset': offset, 'length': stream[i + 2],
                         'extended': True})
            i += 3
            continue
        toks.append({'kind': 'copy', 'pos': i, 'size': 2,
                     'offset': offset, 'length': length})
        i += 2
    return toks


def rebuild(stream, toks, replacements):
    """Troca texto apenas nos literais; copias/RLE ficam intactas."""
    out = bytearray()
    hits = 0
    for t in toks:
        if t['kind'] == 'lit':
            data = t['data']
            for old, new in replacements:
                if old in data:
                    hits += data.count(old)
                    data = data.replace(old, new)
            out.append(0x80 + len(data))
            out += data
        else:
            out += stream[t['pos']:t['pos'] + t['size']]
    return bytes(out), hits


def patch_text(src, dst, replacements):
    """Substitui textos. `replacements` = [(antigo, novo), ...]."""
    if isinstance(replacements, dict):
        replacements = list(replacements.items())
    pairs = [(o.encode('latin1'), n.encode('latin1')) for o, n in replacements]
    raw = xor(read(src))
    toks = stream_info(raw)
    new_stream, hits = rebuild(raw, toks, pairs)
    if hits:
        write(dst, xor(new_stream))
    return hits


def literals(path):
    return [t['data'] for t in stream_info(xor(read(path)))
            if t['kind'] == 'lit']


def strings(path, minlen=4):
    out = []
    for chunk in literals(path):
        txt = ''.join(chr(c) if 32 <= c < 127 else ' ' for c in chunk)
        out.extend(w for w in txt.split() if len(w) >= minlen)
    return out


def describe(path):
    raw = xor(read(path))
    toks = stream_info(raw)
    counts = {}
    for t in toks:
        counts[t['kind']] = counts.get(t['kind'], 0) + 1
    return {'arquivo': path, 'tamanho': len(raw), 'tokens': counts,
            'coberto': sum(t['size'] for t in toks) == len(raw)}
```

---

## §7 — Formato binário completo

Para quem precisar reimplementar tudo.

### Camada 1: ofuscação

```python
plain = bytes(b ^ 0x9A for b in raw)
```

### Camada 2: tokens

| Primeiro byte | Significado |
|---|---|
| `>= 0x80` | literal de `b - 0x80` bytes |
| `< 0x80` | código de 16 bits, **big-endian** |

Do código de 16 bits:

```
code       = (b0 << 8) | b1
comprimento = (code >> SHIFT) + 1
offset      = code & (W - 1)
```

| Comprimento | Token | Bytes |
|---|---|---|
| `1` | RLE (`valor` no byte seguinte, repetições = `offset`) | 3 |
| `2` | comprimento estendido (tamanho real no byte seguinte) | 3 |
| `>= 3` | cópia do ring buffer | 2 |

> O caso `2` é a pegadinha: **não** é cópia de 2 bytes. Ignorar isso
> dessincroniza o parser e "perde" o final do arquivo.

### Camada 3: estrutura do documento

```
ENDPREVIEW
BEGIN_FS          <- esquema hidráulico
  GRAPHRECT O0 ...
  S <prop> <tipo> <valor>
  F <prop> <tipo> <valor>
  D <valor>
END_FS
ENDCT
BEGINDSD          <- escada elétrica / GRAFCET
  DG ...
  SYM ...
ENDDSD
```

Formatos de linha:

```
" F %s x_pos I %ld"       " F %s y_pos I %ld"
" F %s stroke I %ld"      " F %s length I %ld"
" F %s f_offset_x I %ld"  " F %s f_offset_y I %ld"
" F %s only_top_rule B %s" " F %s both_sides_rule B %s"
" S %s B F", " D %ld"
"DG %ld %ld %s ..."       "POS %04ld %04ld %lf 1"
"PID %s", "CREATOR %s"
"DSD %ld", "BEGINDSD", "ENDDSD"
```

### Biblioteca de símbolos

`D:\Hydraulic\sym\<categoria>\<sub>\*.ct`, com `pos.txt` por pasta.

Categorias principais:

```
sym/hyd/actuator/     cilindros (a1 = dupla ação)
sym/hyd/valve/        válvulas
sym/hyd/supply/       bomba, tanque, filtro, acumulador
sym/hyd/sensor/       manômetro, pressostato
sym/electr/switch/    chaves (Manually, Proximit, General)
sym/electr/relay/     relés
sym/electr/sensor/    sensores elétricos
sym/electr/ladder/    contatos de escada
sym/grafcet/          step, transition, action, sub, sync
```

---

## §8 — Limites e por quê

O `.ct` é LZ77 sem cabeçalho, multi-bloco. Escrever do zero exigiria
reimplementar o compressor (`0x41A600` / `0x41A4E0` em `fl_sim_h.exe`) com
sincronização perfeita do ring buffer em cada *flush* de bloco.

Tentativas de regravar o texto completo produzem `Unknown file format` porque
parte do conteúdo mora nos tokens de cópia (não nos literais).

**Caminho para fechar a lacuna** (trabalho futuro):

1. Depuração dinâmica: breakpoint em `0x41B8FD`, despejar `[0x5d1db0]`.
2. Reimplementar o compressor e validar round-trip byte a byte.
3. Automatizar a UI do FluidSIM (colar símbolos e salvar).

Mapa de endereços úteis:

| Endereço | Papel |
|---|---|
| `0x41AEC0` | aplica/remove XOR |
| `0x41A600` | compressor |
| `0x41A4E0` | busca de match LZ |
| `0x41B3A8` | laço do descompressor |
| `0x41B5F7` | leitura do código de 16 bits |
| `0x41B740` | tratamento de `length == 1` |
| `0x41B786` | cópia ring → saída |
| `0x41B83D` | flush final |

| Global | Nome |
|---|---|
| `0x5c544c` | chave XOR (0x9A) |
| `0x5c5454` | `W` |
| `0x5c5458` | `SHIFT` |
| `0x5d1d98` | ring buffer |
| `0x5d1db0` | saída |

---
