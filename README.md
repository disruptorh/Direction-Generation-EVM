# 🧭 EVM Address Generator

Generador de direcciones EVM (Ethereum) que implementa la cadena criptográfica
completa desde la seed phrase hasta la dirección pública: **BIP-39 → BIP-32 →
BIP-44 → secp256k1 → Keccak-256 → EIP-55**. Tool offline, con GUI y con backend
CUDA.

<p align="center">
  <a href="https://github.com/disruptorh/Direction-Generation-EVM/releases/latest/download/direcction-generator">
    <img alt="Descargar" src="https://img.shields.io/badge/%E2%AC%87%20Download-latest%20release-2f6feb?style=for-the-badge&logo=github&logoColor=white">
  </a>
  <a href="https://github.com/disruptorh/Direction-Generation-EVM/releases/latest">
    <img alt="Versiones" src="https://img.shields.io/github/v/release/disruptorh/Direction-Generation-EVM?label=release&style=flat&logo=github&logoColor=white">
  </a>
  <a href="./LICENSE">
    <img alt="Licencia" src="https://img.shields.io/badge/licencia-Apache--2.0-blue?style=flat">
  </a>
</p>

---

## 📥 Descarga rápida

El asset es un ejecutable Linux x86-64 **sin extensión**, empaquetado con
PyInstaller (`onefile`): incluye su propio intérprete de Python, `tkinter` y
`pycryptodome`, así que en la máquina destino no hace falta instalar nada.
Necesita un servidor gráfico X11/Wayland, porque abre una ventana.

```bash
# 1. Descargar el binario de la última release
curl -L -o direcction-generator https://github.com/disruptorh/Direction-Generation-EVM/releases/latest/download/direcction-generator

# 2. Dar permiso de ejecución y arrancar
chmod +x direcction-generator
./direcction-generator
```

---

## ⚠️ Dos programas independientes

El repositorio contiene **dos herramientas separadas que no se llaman entre
sí**. El `.py` no invoca al binario CUDA (no usa `subprocess`, ni `os.system`,
ni menciona `nvcc`): para usar la GPU hay que compilar el `.cu` aparte y
ejecutarlo por separado.

| Fichero | Qué es | Aceleración |
|---|---|---|
| `direcction-generator.py` (470 líneas) | GUI en Tkinter con dos pestañas (lote / frase individual) | **CPU, Python puro**. Solo usa `pycryptodome` para Keccak-256; la aritmética de secp256k1 (`_point_add`, `_point_mul`, `_modinv`) está implementada en el propio script |
| `direcction-generator.cu` (1123 líneas) | CLI por separado para NVIDIA GPU | **CUDA**. El kernel `compute_addresses_kernel` hace la multiplicación escalar y el Keccak-256 en paralelo, con 256 hilos por bloque |

El binario publicado en la release es **solo la GUI**: es un bundle PyInstaller
del `.py` que no incluye ningún binario CUDA.

---

## 🚀 Uso rápido

### GUI (Python, CPU)

```bash
python3 direcction-generator.py
```

- **Pestaña "Desde Archivo (Lote)"**: elige un `.txt` con una seed phrase por
  línea, indica cuántas direcciones derivar por frase (`i=0..n-1`, por defecto
  1) y pulsa **Derivar Lote → direcctions.txt**.
- **Pestaña "Frase Individual"**: pega 12 o 24 palabras y pulsa **Generar 1
  Dirección**.
- Ambas aceptan una *Secret Key / Passphrase* opcional (el passphrase BIP-39).
- Abajo hay una consola de registro con el progreso.

### CLI (CUDA)

```text
./direcction-generator <input.txt> <output.txt> <num_addresses> [passphrase]
```

Es la firma real de `main()` en `direcction-generator.cu:995`. Con menos de 4
argumentos imprime el uso y devuelve 1. `num_addresses` está limitado a un rango
**entre 1 y 10000** por frase. El passphrase es opcional.

Ejemplo ejecutable tal cual: un fichero de entrada con una seed phrase por línea,
100 direcciones por frase y sin passphrase.

```bash
printf 'abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about\n' > seeds.txt && ./direcction-generator seeds.txt addresses.txt 100
```

La derivación es fija en `m/44'/60'/0'/0/i` (BIP-44 para Ethereum, cuenta 0).

Ejemplo ejecutable tal cual, con una frase de prueba:

```bash
# 1. Crear un fichero de entrada con una frase por línea
printf 'test test test test test test test test test test test junk\n' > frases.txt

# 2. Derivar 5 direcciones con el backend CUDA
./direcction-generator frases.txt direcciones.txt 5

# 3. Ver el resultado
cat direcciones.txt
```

### Salida

La GUI **no** escribe en el directorio actual: escribe en un directorio
compartido con el resto de la suite de herramientas.

```text
~/Escritorio/seed-tools-txt/direcctions.txt
```

Usa `~/Desktop/seed-tools-txt` si no existe `Escritorio`. El directorio se crea
solo al arrancar la GUI.

Formato de cada línea que escribe la GUI:

```text
<seed phrase> | m/44'/60'/0'/0/i | 0x<clave privada> | 0x<dirección EIP-55>
```

El CLI escribe un informe por frase con cabecera, la ruta y una tabla de índice /
clave privada / dirección:

```text
═══ Frase #1 (12w) ═══
<seed phrase>
Ruta: m/44'/60'/0'/0/i

Idx  Clave Privada                                                       Dirección
────────────────────────────────────────────────────────────────────────
0    0x<clave privada>                                                   0x<dirección EIP-55>
```

---

## 📦 Compilar desde código

### Requisitos

Para la GUI:

- **Python 3.x** con `tkinter`.
- `pycryptodome` — **obligatoria**: el script importa `Crypto.Hash.keccak` y, si
  falla, imprime el error y hace `sys.exit(1)`. Todo lo demás es biblioteca
  estándar (`os`, `sys`, `hashlib`, `hmac`, `struct`, `tkinter`, `tempfile`,
  `pathlib`); no usa `argparse`.

```bash
# 1. Instalar la única dependencia externa
pip install pycryptodome
```

Para el backend CUDA, además:

- GPU NVIDIA con Compute Capability ≥ 6.0.
- CUDA Toolkit con `nvcc` en el `PATH`.
- OpenSSL (solo las cabeceras; el binario ya enlaza `-lcrypto`).

### Clonar

```bash
# 2. Clonar el repositorio
git clone https://github.com/disruptorh/Direction-Generation-EVM.git
cd Direction-Generation-EVM
```

### Compilar

**No hay Makefile ni script de build en el repositorio.** El `.cu` se compila a
mano:

```bash
# 3. Compilar el CLI CUDA
nvcc -O3 -arch=sm_60 direcction-generator.cu -o direcction-generator -lcrypto
```

Flags y por qué están ahí:

- `-O3`: optimización. El kernel hace la parte pesada (multiplicación escalar en
  secp256k1 + Keccak-256) para todas las direcciones en paralelo.
- `-arch=sm_60`: floor de Compute Capability 6.0. **Ajústalo a tu GPU**: por
  ejemplo `sm_61`, `sm_75`, `sm_86`.
- `-lcrypto`: obligatorio. El `.cu` incluye `<openssl/evp.h>` y
  `<openssl/hmac.h>` y usa `PKCS5_PBKDF2_HMAC` (BIP-39) y `HMAC(EVP_sha512())`
  (BIP-32) en CPU. Sin ese flag falla el enlazado.

Para elegir `-arch` sin adivinar:

```bash
# Ver la Compute Capability de tu GPU y usarla como su_<major><minor>
nvidia-smi --query-gpu=name,compute_cap --format=csv
```

### Ejecutar los tests

```bash
# No hay tests automatizados en el repositorio
```

No hay suite de tests. La verificación cruzada disponible es contrastar las dos
implementaciones: si la GUI y el CLI derivan la misma dirección para la misma
frase, ambas implementan bien BIP-32/44.

### Empaquetar la GUI con PyInstaller (opcional)

Hay tres ficheros `.spec` en el árbol de trabajo, **pero están ignorados por
git** (`.gitignore` contiene `*.spec`), así que no están versionados. Todos
parten de `direcction-generator.py` y solo se diferencian en qué binario CUDA
empaquetan y en el nombre de salida:

| Spec | Binario CUDA que empaqueta | Nombre de salida | Consola |
|---|---|---|---|
| `direcction-generator.spec` | ninguno (`binaries=[]`) | `direcction-generator` | `console=False` (GUI pura) |
| `direction-generator.spec` | `direcction-generator-bin` | `direction-generator` | `console=False` |
| `direction-generator-debug.spec` | `direcction-generator` | `direction-generator-debug` | `console=True` |

Es decir: el primero es el que corresponde al asset publicado; los otros dos
necesitan que dejes el binario CUDA compilado en la raíz con el nombre exacto
que indica la columna (`direcction-generator-bin` o `direcction-generator`), y el
`-debug` además muestra la salida por consola.

```bash
# 4. Instalar PyInstaller
pip install pyinstaller

# 5a. Empaquetar solo la GUI (sin binario CUDA)
pyinstaller --clean direcction-generator.spec

# 5b. Empaquetar GUI + CLI CUDA: compila antes el .cu con el nombre que pide el spec
nvcc -O3 -arch=sm_60 direcction-generator.cu -o direcction-generator-bin && pyinstaller --clean direction-generator.spec
```

Ojo con la doble `c`: el fuente y el `.spec` principal se llaman
`direcction-*`, mientras que `direction-generator.spec` y
`direction-generator-debug.spec` llevan una sola `c`. Son nombres reales del
repositorio, no un error de este README.

---

## 🧰 Comandos útiles / Opciones

### CLI CUDA

| Argumento | Obligatorio | Qué hace |
|---|---|---|
| `input.txt` | sí | Fichero con una seed phrase por línea. Se ignoran las líneas vacías y se recorta el espacio. |
| `output.txt` | sí | Fichero de salida. Se sobrescribe (`std::ofstream` en modo `trunc`). |
| `num_addresses` | sí | Direcciones por frase. Rango válido: 1–10000. |
| `passphrase` | no (4º argumento) | Passphrase BIP-39. Si se omite, cadena vacía. |

Mensajes del propio programa:

```text
Uso: ./direcction-generator <input.txt> <output.txt> <num_addresses> [passphrase]
Error: num_addresses debe ser entre 1 y 10000
Error: no se pudo abrir <fichero>
Error: archivo vacio
```

El progreso va por `stderr` (`Frases: N, Direcciones por frase: M`,
`Procesando frase #N (12w)...`, una línea por dirección y
`Archivo generado: <fichero>`), así que puedes redirigir solo la salida de datos.

```bash
# Separar progreso (stderr) y resultado (stdout del programa → fichero aparte)
./direcction-generator frases.txt direcciones.txt 5 2> progreso.log
```

---

## 🗂️ Estructura del proyecto

```text
.
├── LICENSE                     # Apache-2.0
├── bip39.txt                   # Wordlist BIP-39 en inglés (2048 palabras)
├── direcction-generator.py     # GUI Tkinter (CPU, Python puro + pycryptodome)
├── direcction-generator.cu     # CLI CUDA (derivación BIP-32/44 en CPU, resto en GPU)
├── direcction-generator.spec       # PyInstaller: GUI pura (asset publicado)
├── direction-generator.spec        # PyInstaller: GUI + CUDA (direcction-generator-bin)
├── direction-generator-debug.spec  # PyInstaller: GUI + CUDA, con consola
├── build/                      # salida de PyInstaller (ignorado por git)
└── dist/                       # ejecutables generados (ignorado por git)
```

`bip39.txt` es la wordlist estándar de 2048 palabras. **Ninguno de los dos
programas la lee**: ninguno de los dos fuente la referencia por nombre, y es la
razón por la que no hay validación de checksum (ver Seguridad).

---

## 🔐 Seguridad

- **Ambas herramientas escriben la seed phrase en claro en el fichero de
  salida**, junto a la clave privada. Ese archivo es tan sensible como la propia
  frase: bórralo o cifralo después de usarlo.
- **No se valida el checksum BIP-39.** Ni el Python ni el CUDA comprueban que la
  frase tenga 12/24 palabras válidas ni que el checksum cuadre; derivan
  silenciosamente de cualquier texto, aplicando PBKDF2-HMAC-SHA512 sobre la
  cadena tal cual. Solo importa si vas a barrel tu herramienta con datos de
  terceros.
- El passphrase se pasa como argumento en la línea de comandos del CLI, donde
  queda visible en `ps` y en el historial del shell. Usa la GUI si te importa.
- La GUI escribe siempre en un directorio **predecible** y con **nombre fijo**
  (`direcctions.txt`), en el escritorio. En un equipo compartido, eso expone las
  claves a cualquier otro usuario de la sesión: revisa y borra el fichero.
- Herramienta offline por diseño. Para máxima seguridad, úsala en una máquina
  sin red y nunca compartas seed phrases ni claves privadas.

---

## 📄 Licencia

Apache-2.0 — ver [LICENSE](LICENSE).
