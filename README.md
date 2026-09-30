# 🧭 EVM Address Generator

Generador de direcciones EVM (Ethereum) que implementa la cadena criptográfica
completa desde la seed phrase hasta la dirección pública: **BIP-39 → BIP-32 →
BIP-44 → secp256k1 → Keccak-256 → EIP-55**.

![Licencia](https://img.shields.io/badge/License-Apache--2.0-yellow.svg)

## ⚠️ Dos programas independientes

Este repositorio contiene **dos herramientas separadas que no se llaman entre
sí**. La documentación anterior sugería que la GUI de Python aceleraba la
derivación con el backend CUDA; no es así.

| Fichero | Qué es | Aceleración |
|---|---|---|
| `direcction-generator.py` (470 líneas) | GUI en Tkinter con dos pestañas (lote / frase individual) | **CPU, Python puro**. Solo usa `pycryptodome` para Keccak-256; la aritmética de secp256k1 (`_point_add`, `_point_mul`, `_modinv`) está implementada en el propio script |
| `direcction-generator.cu` (1123 líneas) | CLI por separado para NVIDIA GPU | **CUDA**. El kernel `compute_addresses_kernel` hace la mult. escalar y el Keccak-256 en paralelo, 256 hilos/bloque |

No hay `Makefile`, ni `subprocess`, ni referencia a `nvcc` en el Python: para
usar la GPU hay que compilar el `.cu` a mano y ejecutarlo por separado.

## Requisitos

- **Python 3.x** con `tkinter` (interfaz gráfica)
- `pycryptodome` — obligatorio: el script aborta al arrancar si no está

  ```bash
  pip install pycryptodome
  ```

- **Solo para el backend CUDA:** GPU NVIDIA, Compute Capability ≥ 6.0 y el
  CUDA Toolkit con `nvcc` en el `PATH`.

## Compilación del backend CUDA

```bash
nvcc -O3 -arch=sm_60 direcction-generator.cu -o direcction-generator -lcrypto
```

(`-lcrypto` es necesario: PBKDF2-HMAC-SHA512 y HMAC-SHA512 salen de OpenSSL.)

Ajusta `-arch` a tu GPU (`sm_61`, `sm_75`, `sm_86`, …). **No hay script de
build ni Makefile en el repositorio**; este comando hay que correrlo a mano.

## Uso

### GUI (Python, CPU)

```bash
python3 direcction-generator.py
```

- **Pestaña "Desde Archivo (Lote)"**: elige un `.txt` con una seed phrase por
  línea, indica cuántas direcciones derivar por frase (`i=0..n-1`, por defecto
  1) y pulsa **Derivar Lote**.
- **Pestaña "Frase Individual"**: pega 12 o 24 palabras y pulsa **Generar 1
  Dirección**.
- Ambas aceptan una *Secret Key / Passphrase* opcional (el passphrase BIP-39).
- Abajo hay una consola de registro con el progreso.

### CLI (CUDA)

```bash
./direcction-generator <input.txt> <output.txt> <num_addresses> [passphrase]
```

- Límite: entre 1 y 10000 direcciones por frase.
- Derivación fija en `m/44'/60'/0'/0/i` (BIP-44 para Ethereum, cuenta 0).

### Rutas y salida

La GUI escribe en un directorio compartido con el resto de la suite, no en el
directorio actual:

```
~/Escritorio/seed-tools-txt/direcctions.txt
```

(cambia a `~/Desktop/seed-tools-txt` si no existe `Escritorio`).

Formato de cada línea:

```
<seed phrase> | m/44'/60'/0'/0/i | 0x<clave privada> | 0x<Dirección EIP-55>
```

## Estándares implementados

- **BIP-39** — mnemónico → seed (PBKDF2-HMAC-SHA512, 2048 iteraciones,
  salt `"mnemonic" + passphrase`)
- **BIP-32** — wallets jerárquicas deterministas (HMAC-SHA512, derivación
  *hardened* `0x00 || k_par || ser32(i)` y *normal* `serP(k_par) || ser32(i)`)
- **BIP-44** — `m/44'/60'/0'/0/i`, las tres derivaciones *hardened* en CPU
- **secp256k1** — parámetros de la curva y aritmética de punto
- **Keccak-256** — hash de Ethereum; la dirección son los 20 últimos bytes del
  hash de la pubkey sin comprimir `x || y`
- **EIP-55** — codificación con checksum en mayúsculas/minúsculas

## Notas de seguridad

- **Ambas herramientas escriben la seed phrase en claro en el fichero de
  salida**, junto a la clave privada. Ese archivo es tan sensible como la propia
  frase: bórralo o cifralo después de usarlo.
- **No se valida el checksum BIP-39.** Ni el Python ni el CUDA comprueban que la
  frase tenga 12/24 palabras válidas ni que el checksum cuadre; derivan
  silenciosamente de cualquier texto. Solo importa si vas a barrel tu
  herramienta con datos de terceros.
- El passphrase se pasa como argumento en la línea de comandos del CLI, donde
  queda visible en `ps` y en el historial del shell. Usa la GUI si te importa.
- Herramienta offline por diseño. Para máxima seguridad, úsala en una máquina
  sin red y nunca compartas seed phrases ni claves privadas.

## Tests

No hay tests automatizados. Ambas implementaciones se pueden contrastar entre
sí como verificación cruzada: si derivan la misma dirección para la misma
frase, ambas implementan bien BIP-32/44.

## Licencia

Apache-2.0 — ver [LICENSE](LICENSE).
