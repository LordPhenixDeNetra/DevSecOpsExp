# Recueil de batailles techniques

> Un journal des bugs, des impasses et des victoires qui ont façonné mon parcours de dev.

Ce dépôt contient un recueil personnel d'expériences de DevSecOps : chaque entrée raconte un problème technique rencontré, comment il a été investigué, résolu, et quelle leçon en a été tirée. Le contenu est maintenu **en double format** : Markdown pour l'édition rapide et LaTeX pour une sortie PDF mise en page.

## 📁 Arborescence

```
DEV_SEC_OPS_EXP/
├── dev_sec_ops_exp.md    # Contenu principal (source Markdown)
├── main.tex              # Version LaTeX compilable → PDF
├── exemple.md            # Gabarit Markdown pour une nouvelle entrée
├── exemple.tex           # Gabarit LaTeX / snippet de référence
├── recueil.docx          # Export Word (si nécessaire)
├── .gitignore
└── README.md
```

## ✍️ Structure d'une entrée

Chaque entrée suit une narration en 6 actes, inspirée du format AAR (After-Action Review) :

| Section | Rôle |
|---|---|
| **Contexte** | Ce qui était construit, état de l'infrastructure, état d'esprit |
| **Le problème** | Symptôme exact, premier constat, message d'erreur clé |
| **L'enquête** | Pistes explorées, fausses pistes, éléments contradictoires |
| **Le déclic** | L'observation qui a tout expliqué |
| **La résolution** | Étapes concrètes pour corriger, snippets si utiles |
| **Ce que j'en retiens** | Leçons, heuristiques, réflexes gagnés |
| *Tags* | Mots-clés pour retrouver l'entrée (docker, dokploy, réseau, etc.) |

## ➕ Ajouter une entrée

1. Copier le gabarit de [exemple.md](exemple.md)
2. L'insérer à la fin de [dev_sec_ops_exp.md](dev_sec_ops_exp.md), juste avant la dernière section
3. Reproduire la même insertion dans [main.tex](main.tex) (adapter les accents, blocs `lstlisting`, `\tag{...}`)

## 🧾 Compiler la version LaTeX

Pré-requis : une distribution TeX (TeX Live, MiKTeX, MacTeX) avec les paquets courants (`geometry`, `titlesec`, `listings`, `microtype`, `csquotes`, `babel[french]`, `hyperref`, `lmodern`).

```bash
# Compilation standard (pdflatex, 2 passes pour la TOC)
pdflatex main.tex
pdflatex main.tex

# Ou avec latexmk (recommandé — gère les passes automatiquement)
latexmk -pdf main.tex
```

Le PDF généré `main.pdf` est ignoré par git (voir `.gitignore`).

## 🎨 Mise en page

- Classe `article`, 11pt, A4, marges 2.8 cm, interligne 1.5
- Police : Latin Modern avec `microtype`
- Couleurs : bleu accent (titres, liens), olive (tags)
- Blocs de code cadrés sur fond gris
- Séparateurs visuels entre chaque entrée

## 📜 Licence

Usage privé / personnel. Ce recueil documente des expériences vécues — libre à chacun de s'en inspirer.
