# 🧬 Lysozyme in Water — GROMACS Tutorial

Tutorial de **Dinâmica Molecular (MD)** utilizando o **GROMACS**, com foco na simulação da proteína **lysozyme (lisozima) em meio aquoso**.

O objetivo deste repositório é apresentar, de forma prática e reprodutível, as principais etapas necessárias para preparar, executar e analisar uma simulação de dinâmica molecular de uma proteína em água.

---

## 📌 Sobre o tutorial

Neste tutorial, a estrutura da lisozima é preparada para uma simulação atomística em água, passando pelas principais etapas de um workflow de **Molecular Dynamics**:

```text
Estrutura PDB
     │
     ▼
Preparação do sistema
     │
     ▼
Definição do campo de força
     │
     ▼
Solvatação
     │
     ▼
Adição de íons
     │
     ▼
Minimização de energia
     │
     ▼
Equilibração
 ┌───┴───┐
 ▼       ▼
NVT     NPT
 └───┬───┘
     ▼
Produção MD
     │
     ▼
Análise da trajetória
```

---

## 🔬 Sistema estudado

**Proteína:** Lysozyme
**Meio:** Água
**Método:** Dinâmica Molecular clássica
**Software:** GROMACS
**Estrutura:** Protein Data Bank (PDB)

O sistema pode ser utilizado para estudar propriedades estruturais e dinâmicas da proteína, como:

* RMSD — *Root Mean Square Deviation*
* RMSF — *Root Mean Square Fluctuation*
* Raio de giro (*Radius of Gyration*)
* Ligações de hidrogênio
* Distância e estrutura da proteína
* Estabilidade estrutural ao longo da simulação

---

## ⚙️ Requisitos

Antes de começar, certifique-se de possuir:

* Linux
* GROMACS
* Python 3
* Git
* Ferramentas para visualização molecular, como:

  * VMD
  * PyMOL
  * UCSF ChimeraX

Para verificar a instalação do GROMACS:

```bash
gmx --version
```

---

## 📂 Estrutura do repositório

```text
.
├── README.md
│
├── pdb/
│   └── lysozyme.pdb
│
├── topology/
│   ├── topol.top
│   └── *.itp
│
├── mdp/
│   ├── minim.mdp
│   ├── nvt.mdp
│   ├── npt.mdp
│   └── md.mdp
│
├── scripts/
│   └── ...
│
├── analysis/
│   ├── rmsd/
│   ├── rmsf/
│   ├── gyration/
│   └── hydrogen_bonds/
│
└── results/
    └── ...
```

> A estrutura pode ser modificada conforme o desenvolvimento do tutorial.

---

# 🚀 Workflow

## 1. Obtenção da estrutura

A primeira etapa consiste em obter a estrutura cristalográfica da lisozima no formato `.pdb`.

O arquivo deve ser colocado no diretório:

```text
pdb/
```

Exemplo:

```bash
mkdir -p pdb
```

---

## 2. Preparação da estrutura

A estrutura inicial deve ser preparada para a simulação, verificando:

* presença de moléculas de água;
* ligantes ou moléculas não desejadas;
* resíduos incompletos;
* átomos ausentes;
* nomenclatura dos resíduos;
* protonação.

---

## 3. Geração da topologia

A topologia da proteína é gerada utilizando o `pdb2gmx`:

```bash
gmx pdb2gmx -f pdb/lysozyme.pdb -o lysozyme_processed.gro -water tip3p
```

Durante esse processo, o GROMACS solicitará a escolha de um **campo de força**.

---

## 4. Definição da caixa de simulação

Uma caixa de simulação é criada ao redor da proteína:

```bash
gmx editconf \
    -f lysozyme_processed.gro \
    -o lysozyme_box.gro \
    -c \
    -d 1.0 \
    -bt cubic
```

O parâmetro `-d` define a distância mínima entre a proteína e as bordas da caixa.

---

## 5. Solvatação

A caixa é preenchida com moléculas de água:

```bash
gmx solvate \
    -cp lysozyme_box.gro \
    -cs spc216.gro \
    -o lysozyme_solvated.gro \
    -p topol.top
```

A escolha do modelo de água deve ser consistente com o campo de força utilizado.

---

## 6. Adição de íons

Os íons são adicionados para neutralizar a carga do sistema e, se desejado, reproduzir uma determinada força iônica.

Primeiro:

```bash
gmx grompp \
    -f mdp/ions.mdp \
    -c lysozyme_solvated.gro \
    -p topol.top \
    -o ions.tpr
```

Depois:

```bash
gmx genion \
    -s ions.tpr \
    -o lysozyme_ions.gro \
    -p topol.top \
    -pname NA \
    -nname CL \
    -neutral
```

---

# ⚡ 7. Minimização de energia

Antes da dinâmica, o sistema passa por uma etapa de **Energy Minimization** para remover contatos estéricos e configurações energeticamente desfavoráveis.

```bash
gmx grompp \
    -f mdp/minim.mdp \
    -c lysozyme_ions.gro \
    -p topol.top \
    -o em.tpr
```

Execução:

```bash
gmx mdrun -deffnm em
```

Verificação da energia potencial:

```bash
gmx energy -f em.edr -o potential.xvg
```

---

# 🌡️ 8. Equilibração NVT

A etapa NVT mantém:

* número de partículas constante;
* volume constante;
* temperatura constante.

```bash
gmx grompp \
    -f mdp/nvt.mdp \
    -c em.gro \
    -r em.gro \
    -p topol.top \
    -o nvt.tpr
```

```bash
gmx mdrun -deffnm nvt
```

A temperatura pode então ser analisada utilizando:

```bash
gmx energy -f nvt.edr -o temperature.xvg
```

---

# 💧 9. Equilibração NPT

Na etapa NPT, a pressão também é controlada.

```bash
gmx grompp \
    -f mdp/npt.mdp \
    -c nvt.gro \
    -r nvt.gro \
    -p topol.top \
    -o npt.tpr
```

```bash
gmx mdrun -deffnm npt
```

Podem ser analisadas, por exemplo:

* temperatura;
* pressão;
* densidade;
* volume.

---

# 🧬 10. Produção da dinâmica molecular

Após a equilibração, inicia-se a etapa de produção da simulação.

```bash
gmx grompp \
    -f mdp/md.mdp \
    -c npt.gro \
    -t npt.cpt \
    -p topol.top \
    -o md.tpr
```

Execução:

```bash
gmx mdrun -deffnm md
```

Os principais arquivos gerados incluem:

```text
md.xtc   → trajetória
md.tpr   → arquivo de entrada
md.edr   → dados de energia
md.log   → log da simulação
md.gro   → estrutura final
md.cpt   → checkpoint
```

---

# 📊 11. Análise da trajetória

## RMSD

O RMSD permite avaliar o desvio estrutural da proteína em relação a uma estrutura de referência.

```bash
gmx rms \
    -s md.tpr \
    -f md.xtc \
    -o analysis/rmsd/rmsd.xvg
```

---

## RMSF

O RMSF permite avaliar a flexibilidade dos resíduos ao longo da trajetória.

```bash
gmx rmsf \
    -s md.tpr \
    -f md.xtc \
    -o analysis/rmsf/rmsf.xvg
```

---

## Raio de giro

O raio de giro fornece informações sobre a compactação da proteína:

```bash
gmx gyrate \
    -s md.tpr \
    -f md.xtc \
    -o analysis/gyration/gyration.xvg
```

---

## Ligações de hidrogênio

As ligações de hidrogênio podem ser investigadas com:

```bash
gmx hbond
```

Os resultados podem ser utilizados para avaliar interações intramoleculares e proteína–solvente.

---

# 🖥️ Visualização

A trajetória pode ser visualizada utilizando softwares como **VMD**, **PyMOL** ou **ChimeraX**.

Exemplo utilizando VMD:

```bash
vmd md.tpr md.xtc
```

A visualização permite observar alterações conformacionais da proteína durante a dinâmica.

---

# 📈 Análises com Python

Os arquivos `.xvg` gerados pelo GROMACS podem ser processados utilizando Python.

Bibliotecas úteis:

```bash
pip install numpy pandas matplotlib
```

Exemplo:

```python
import numpy as np
import matplotlib.pyplot as plt

data = np.loadtxt("rmsd.xvg", comments=["@", "#"])

time = data[:, 0]
rmsd = data[:, 1]

plt.plot(time, rmsd)

plt.xlabel("Time (ps)")
plt.ylabel("RMSD (nm)")
plt.title("Lysozyme RMSD")

plt.show()
```

---

# 🎯 Objetivos do tutorial

Ao final deste tutorial, o usuário deverá ser capaz de:

* preparar uma proteína para dinâmica molecular;
* construir uma caixa de simulação;
* solvatar o sistema;
* adicionar íons;
* realizar minimização de energia;
* executar equilíbrios NVT e NPT;
* realizar uma simulação de produção;
* visualizar trajetórias;
* analisar propriedades estruturais da proteína;
* utilizar Python para tratamento e visualização dos resultados.

---

## 📚 Conceitos envolvidos

Este tutorial envolve conceitos de:

**Química Computacional · Dinâmica Molecular · Biofísica Computacional · Mecânica Estatística · Simulação Atomística · Análise Estrutural de Proteínas**

---

## 🛠️ Software

* **GROMACS** — Molecular Dynamics
* **VMD** — Visualização molecular
* **Python** — Análise de dados
* **Git/GitHub** — Versionamento

---

## 📖 Referências

* Abraham, M. J. et al. *GROMACS: High performance molecular simulations through multi-level parallelism from laptops to supercomputers*. SoftwareX.
* Documentação oficial do GROMACS.
* Protein Data Bank (PDB).

⭐ Se este tutorial for útil, considere deixar uma estrela no repositório!
