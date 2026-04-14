---
name: doc-generation
description: Write or update IEC 62304-compliant software documentation (SRS, SDD, User Manual) as ReStructuredText with Sphinx + Breathe, producing a PDF suitable for upload into a document management system. Supports creation and updating of diagrams (architecture, sequence, UML, state, ER) with consistent cross-project styling exported to PDF. Trigger when the user asks to "create docs", "write documentation", "update the SRS/SDD/manual", "generate documentation", "add diagrams", "create architecture diagram", or similar.
---

# Documentation Generation Skill (IEC 62304 / Sphinx / PDF)

Create or update RST-based software documentation that compiles to a DMS-ready PDF.
Documents follow IEC 62304 structure (SRS, SDD, User Manual) with consistent style across projects.

## Tools Required

Check availability before starting:

```bash
sphinx-build --version        # Sphinx >= 7
doxygen --version             # for C/C++ projects
breathe-apidoc --version      # Breathe Sphinx extension
pdflatex --version            # for PDF output
plantuml --version            # PlantUML >= 1.2024, native binary preferred (no Java)
```

Install if missing:
```bash
pip install sphinx breathe sphinx-rtd-theme sphinxcontrib-plantuml
sudo apt-get install -y doxygen texlive-latex-extra texlive-fonts-recommended
```

**PlantUML — Linux native binary (no Java required):**
```bash
# Check https://github.com/plantuml/plantuml/releases for the latest version.
# Download a release labelled "-native" or "-full" (GraalVM-compiled, no JVM needed).
# Example for v1.2025.4:
curl -L -o /tmp/plantuml-native.zip \
  "https://github.com/plantuml/plantuml/releases/download/v1.2025.4/plantuml-full-linux-amd64-1.2025.4.zip"
unzip -j /tmp/plantuml-native.zip -d /tmp/plantuml-native
sudo mv /tmp/plantuml-native/plantuml /usr/local/bin/plantuml
chmod +x /usr/local/bin/plantuml
plantuml --version
```

> `apt-get install plantuml` installs the **JVM version** (requires Java). Use the manual download above for the Java-free native binary.

**PlantUML — Windows 11 native binary (no Java required):**
```powershell
# Check https://github.com/plantuml/plantuml/releases for the latest version.
# Download a release labelled "-native" or "-full" for Windows (amd64).
# Extract plantuml.exe, place it in e.g. C:\tools\plantuml\, then add to PATH:
[System.Environment]::SetEnvironmentVariable(
    "PATH",
    "$Env:PATH;C:\tools\plantuml",
    [System.EnvironmentVariableTarget]::User
)
# Open a new terminal and verify:
plantuml --version
```

> `choco install plantuml` installs the **JVM version** (requires Java). Use the manual download above for the Java-free native binary.

## Document Types

| Type | IEC 62304 Ref | Directory |
|------|--------------|-----------|
| Software Requirements Specification (SRS) | §5.2 | `docs/srs/` |
| Software Design Document (SDD) | §5.3–5.4 | `docs/sdd/` |
| User / Operator Manual (UM) | — | `docs/um/` |

## Workflow

Make a todo list for all tasks and work through them one at a time.

### 1. Discover Project Context

Read the following before writing anything:
- `README.md` — project name, purpose, overview
- Source headers in `include/` or `src/` — understand the public API and component structure
- Existing `docs/` directory — never overwrite existing content, only extend it
- `CHANGELOG` or recent git log — current version and recent changes

### 2. Load or Create Document Metadata

Check for `.claude/doc-meta.yaml`. If it exists, read it and use those values throughout.
If it does not exist, create it — infer what you can from the README and source, then ask the user only for values you cannot determine:

```yaml
# .claude/doc-meta.yaml
# Document metadata used by the doc-generation skill.
# Update version and release_date before each new document revision.
project_name: "My Project"
project_abbr: "MPROJ"           # short ID used in document numbers: DOC-MPROJ-SRS-001
company_name: "Acme Corp"
safety_class: "B"               # IEC 62304 Software Safety Class: A, B, or C
confidentiality: "Confidential" # Confidential | Internal | Public
primary_author: "Jane Doe"
version: "1.0"
release_date: "YYYY-MM-DD"
```

### 3. Set Up Sphinx Infrastructure (if not present)

Create the following structure (skip files that already exist):

```
docs/
├── conf.py
├── index.rst              ← root document listing all sub-documents
├── Makefile
├── requirements-docs.txt
├── _plantuml_style.puml   ← shared diagram theme (commit this)
├── Doxyfile               ← C/C++ only
├── _static/
│   └── custom.css
├── _templates/
├── srs/                   ← if generating SRS
│   └── index.rst
├── sdd/                   ← if generating SDD
│   └── index.rst
└── um/                    ← if generating User Manual
    └── index.rst
```

**`docs/requirements-docs.txt`:**
```
sphinx>=7.0
breathe>=4.35
sphinx-rtd-theme>=2.0
sphinxcontrib-plantuml>=0.9
```

**`docs/Makefile`:**
```makefile
SPHINXOPTS    ?=
SPHINXBUILD   ?= sphinx-build
SOURCEDIR     = .
BUILDDIR      = _build

.PHONY: help Makefile latexpdf html doxygen

help:
	@$(SPHINXBUILD) -M help "$(SOURCEDIR)" "$(BUILDDIR)" $(SPHINXOPTS) $(O)

doxygen:
	doxygen Doxyfile

latexpdf html: doxygen Makefile
	@$(SPHINXBUILD) -M $@ "$(SOURCEDIR)" "$(BUILDDIR)" $(SPHINXOPTS) $(O)
	@if [ "$@" = "latexpdf" ]; then echo "PDF: $(BUILDDIR)/latex/*.pdf"; fi
```

**`docs/conf.py`** — substitute all values from `.claude/doc-meta.yaml`:
```python
# Configuration file for Sphinx documentation.
# Values are populated from .claude/doc-meta.yaml by the doc-generation skill.

# -- Project metadata --------------------------------------------------------
project     = "<project_name>"
author      = "<primary_author>"
version     = "<version>"
release     = "<version>"
copyright   = "<release_date>, <company_name>"

# -- General configuration ---------------------------------------------------
extensions = [
    "breathe",
    "sphinx.ext.autosectionlabel",
    "sphinx.ext.todo",
    "sphinx.ext.ifconfig",
    "sphinxcontrib.plantuml",
]
autosectionlabel_prefix_document = True
todo_include_todos = False
exclude_patterns = ["_build", "Thumbs.db", ".DS_Store"]

# -- Breathe (C/C++ API via Doxygen) -----------------------------------------
breathe_projects        = {"<project_abbr>": "_build/doxygen/xml"}
breathe_default_project = "<project_abbr>"

# -- PlantUML diagrams -------------------------------------------------------
import os as _os
_plantuml_style = _os.path.join(_os.path.dirname(__file__), '_plantuml_style.puml')
# Tuple form is safe when paths contain spaces
plantuml = ('plantuml', '-config', _plantuml_style)
plantuml_output_format       = 'png'   # HTML: PNG (change to 'svg' for interactive HTML)
plantuml_latex_output_format = 'png'   # LaTeX/PDF: PNG is most reliable with pdflatex

# -- HTML output -------------------------------------------------------------
html_theme = "sphinx_rtd_theme"

# -- LaTeX / PDF output ------------------------------------------------------
latex_engine = "pdflatex"
latex_documents = [(
    "index",
    "<project_abbr>.tex",
    "<project_name>",
    "<primary_author>",
    "manual",
)]

_doc_number      = "DOC-<PROJECT_ABBR>-001"
_safety_class    = "Software Safety Class <safety_class> (IEC 62304)"
_confidentiality = "<confidentiality>"
_release_date    = "<release_date>"
_company         = "<company_name>"

latex_elements = {
    "papersize": "a4paper",
    "pointsize": "11pt",
    "preamble": r"""
\usepackage{booktabs}
\usepackage{longtable}
\usepackage{array}
\usepackage{fancyhdr}

\newcommand{\docnumber}{""" + _doc_number + r"""}
\newcommand{\docsafetyclass}{""" + _safety_class + r"""}
\newcommand{\docconfidentiality}{""" + _confidentiality + r"""}

\pagestyle{fancy}
\fancyhead[L]{\small\docnumber}
\fancyhead[R]{\small\docconfidentiality}
\fancyfoot[C]{\small Page \thepage}
""",
    "maketitle": r"""
\begin{titlepage}
  \centering
  \vspace*{2cm}
  {\Huge \textbf{""" + project + r"""} \par}
  \vspace{0.5cm}
  {\Large \textit{""" + "<document_full_name>" + r"""} \par}
  \vspace{2.5cm}
  \begin{tabular}{@{}ll@{}}
    \textbf{Document Number:}   & \docnumber \\[4pt]
    \textbf{Version:}           & """ + version + r""" \\[4pt]
    \textbf{Date:}              & """ + _release_date + r""" \\[4pt]
    \textbf{Author:}            & """ + author + r""" \\[4pt]
    \textbf{Classification:}    & \docconfidentiality \\[4pt]
    \textbf{Safety Class:}      & \docsafetyclass \\
  \end{tabular}
  \vspace{2.5cm}
  {\large """ + _company + r""" \par}
  \vfill
  \small\textit{This document is subject to change control.
  The approved version is held in the document management system.}
\end{titlepage}
""",
}
```

**`docs/_plantuml_style.puml`** — shared cross-project diagram theme:
```plantuml
' Shared PlantUML style — committed to docs/_plantuml_style.puml
' Applied automatically via -config flag in conf.py.
' Do NOT add !theme or skinparam to individual diagrams.
!theme spacelab
skinparam defaultFontName  "Helvetica Neue, Helvetica, Arial, sans-serif"
skinparam defaultFontSize  13
skinparam dpi               150
skinparam shadowing         false
skinparam roundcorner       8
skinparam BackgroundColor   White
skinparam ArrowColor        #2D5EA8
skinparam ArrowFontColor    #1A1A2E
skinparam BorderColor       #2D5EA8
skinparam NoteBackgroundColor #FFFDE7
skinparam NoteBorderColor   #F0C040
skinparam NoteFontColor     #1A1A2E
skinparam SequenceLifeLineBorderColor        #2D5EA8
skinparam SequenceActorBorderColor           #2D5EA8
skinparam SequenceActorBackgroundColor       #2D5EA8
skinparam SequenceActorFontColor             #FFFFFF
skinparam SequenceParticipantBorderColor     #2D5EA8
skinparam SequenceParticipantBackgroundColor #E8F0FB
skinparam ClassBorderColor            #2D5EA8
skinparam ClassBackgroundColor        #E8F0FB
skinparam ClassHeaderBackgroundColor  #2D5EA8
skinparam ClassFontColor              #FFFFFF
skinparam ComponentBorderColor        #2D5EA8
skinparam ComponentBackgroundColor    #E8F0FB
skinparam DatabaseBorderColor         #2D5EA8
skinparam DatabaseBackgroundColor     #E8F0FB
```

> `spacelab` provides clean Bootstrap-derived spacing; the `skinparam` block replaces its accent colour with a consistent corporate blue (`#2D5EA8`) that matches the document header colour defined in `latex_elements`. Copy this file unchanged to every project — all diagrams across projects will share the same look.

**`docs/index.rst`** (root, lists all documents):
```rst
<project_name> — Documentation
================================

.. toctree::
   :maxdepth: 1
   :caption: Documents

   srs/index
   sdd/index
   um/index
```
Only include toctree entries for document types being generated.

### 4. Configure Doxygen (C/C++ projects only)

Create `docs/Doxyfile` if not present:
```
PROJECT_NAME     = "<project_name>"
OUTPUT_DIRECTORY = _build/doxygen
INPUT            = ../src ../include
RECURSIVE        = YES
GENERATE_XML     = YES
GENERATE_HTML    = NO
GENERATE_LATEX   = NO
EXTRACT_ALL      = YES
QUIET            = YES
```

Note: `INPUT` paths are relative to the `docs/` directory where Doxygen is run.

### 5. Write RST Document Content

**Style rules — enforce without exception:**
- Title underlines: `=` (H1 document title), `-` (H2 section), `~` (H3 subsection), `^` (H4)
- Tables: always `.. list-table::` with `:widths:` and `:header-rows: 1`
- Cross-references: `:ref:`` with `autosectionlabel` labels — never bare section names
- Safety notes: `.. warning::` for safety-critical items, `.. note::` for informational, `.. important::` for mandatory compliance items
- No placeholders: never leave "TBD" or empty sections — write real content from project context

#### Standard Preamble (every document's index.rst)

```rst
.. _<doctype>_document:

<Project Name> — <Document Full Name>
======================================

.. list-table:: Document Information
   :widths: 35 65
   :header-rows: 0

   * - Document Number
     - DOC-<PROJECT_ABBR>-<DOCTYPE>-001
   * - Version
     - <version>
   * - Date
     - <release_date>
   * - Author
     - <primary_author>
   * - Software Safety Class
     - Class <safety_class> (IEC 62304)
   * - Classification
     - <confidentiality>

.. toctree::
   :maxdepth: 2
   :numbered:

   scope
   applicable_documents
   definitions
   <document-type-specific sections — see below>
   revision_history
```

#### Mandatory Sections (all document types)

**`scope.rst`**
```rst
Scope
-----

<Write based on README and source analysis.>
<State: what software is covered, what is out of scope, the IEC 62304 safety class and rationale.>
```

**`applicable_documents.rst`**
```rst
Applicable Documents
--------------------

.. list-table::
   :widths: 12 58 30
   :header-rows: 1

   * - Ref
     - Title
     - Identifier / Version
   * - [IEC62304]
     - IEC 62304: Medical device software — Software life cycle processes
     - Ed. 1.1, 2015
   * - <add project-specific referenced docs>
     -
     -
```

**`definitions.rst`**
```rst
Definitions and Abbreviations
------------------------------

.. glossary::
   :sorted:

   IEC 62304
      International standard for medical device software life cycle processes.

   SRS
      Software Requirements Specification

   SDD
      Software Design Document

   <add project-specific terms inferred from source code and README>
```

**`revision_history.rst`** (append a new row on every update)
```rst
Revision History
----------------

.. list-table::
   :widths: 8 18 22 52
   :header-rows: 1

   * - Rev
     - Date
     - Author
     - Description of Changes
   * - 1.0
     - <release_date>
     - <primary_author>
     - Initial release
```

---

#### SRS-Specific Sections

Add after `definitions.rst`:

```
   software_overview
   requirements/functional
   requirements/performance
   requirements/interfaces
   requirements/safety
```

**`software_overview.rst`** — purpose, operating environment, intended users/roles, assumptions.

**`requirements/functional.rst`** — one requirement per row, use IDs:

```rst
Functional Requirements
~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :widths: 14 71 15
   :header-rows: 1

   * - ID
     - Requirement
     - Priority
   * - SRS-F-001
     - The software shall <verb> <object> <measurable condition>.
     - Mandatory
```

**`requirements/safety.rst`** — safety requirements per IEC 62304 §5.2.2:

```rst
Safety Requirements
~~~~~~~~~~~~~~~~~~~

.. important::

   The following requirements are safety-critical and apply to Class <safety_class> software.

.. list-table::
   :widths: 14 71 15
   :header-rows: 1

   * - ID
     - Requirement
     - Safety Class
   * - SRS-S-001
     - The software shall <safety constraint>.
     - <A/B/C>
```

---

#### SDD-Specific Sections

Add after `definitions.rst`:

```
   architecture
   components/index
   interfaces
   data_models
```

**`architecture.rst`** — system context, decomposition into major subsystems, rationale for key design decisions.

**`components/index.rst`** — toctree referencing one .rst per component:

```rst
Software Components
~~~~~~~~~~~~~~~~~~~

.. toctree::

   component_foo
   component_bar
```

**`components/component_foo.rst`** — per-component description with Breathe API docs:

```rst
Component: FooManager
~~~~~~~~~~~~~~~~~~~~~

<Brief description of responsibility and lifecycle.>

.. doxygenclass:: FooManager
   :project: <project_abbr>
   :members:
   :undoc-members:

.. doxygenfunction:: foo_init
   :project: <project_abbr>
```

---

#### User Manual-Specific Sections

Add after `definitions.rst`:

```
   safety_notices
   system_requirements
   installation
   operation
   troubleshooting
```

**`safety_notices.rst`** — must appear near the front; use `.. warning::` admonitions for each safety-critical notice.

**`installation.rst`** and **`operation.rst`** — use numbered steps (`#.` lists) for procedures; use `.. code-block:: bash` for commands.

---

### 5.5. Create or Update Diagrams

Add diagrams wherever they aid understanding — they are required in the following locations:

| Document | File | Diagram type |
|----------|------|--------------|
| SDD | `architecture.rst` | Component/system overview (always include) |
| SDD | `components/component_*.rst` | Class diagram per non-trivial component |
| SRS | `requirements/interfaces.rst` | Sequence diagram per key interface |
| SDD | `interfaces.rst` | Sequence diagrams for external interfaces |
| SDD | any state-based component | State machine diagram |
| SDD | `data_models.rst` | ER diagram |
| UM | `operation.rst` | Flowchart for multi-step procedures |

**Diagram style rules — enforce without exception:**
- Use `.. uml::` directive exclusively — never embed image files directly
- Every `.. uml::` block must include `:caption: Figure N — <short descriptive title>`
- Use `:scale: 80 %` for comfortable margins in both PDF and HTML
- Keep diagram source inline in the RST file — no separate `.puml` sidecar files
- Do **not** add `!theme` or `skinparam` to individual diagrams — `_plantuml_style.puml` handles all theming automatically via the `-config` flag
- Figure numbers increment per-document, reset at each document type (SRS, SDD, UM)

**RST directive format:**
```rst
.. uml::
   :caption: Figure N — <Title>
   :scale: 80 %

   @startuml
   ' diagram source here — no !theme or skinparam needed
   @enduml
```

---

#### Component / Architecture Diagram
```rst
.. uml::
   :caption: Figure 1 — <SystemName> Component Overview
   :scale: 80 %

   @startuml
   package "External" {
       [Client App] as Client
   }
   package "<SystemName>" {
       [API Layer]      as API
       [Business Logic] as BL
       database "Database" as DB
   }
   Client --> API  : HTTPS
   API    --> BL
   BL     --> DB
   @enduml
```

#### Sequence Diagram
```rst
.. uml::
   :caption: Figure N — <UseCase> Interaction Sequence
   :scale: 80 %

   @startuml
   autonumber
   actor     User
   participant UI
   participant "API Layer" as API
   participant "Database"  as DB

   User -> UI  : Submit request
   UI   -> API : POST /endpoint
   API  -> DB  : INSERT record
   DB  --> API : OK
   API --> UI  : 201 Created
   UI  --> User: Confirmation
   @enduml
```

#### UML Class Diagram
```rst
.. uml::
   :caption: Figure N — <Module> Class Structure
   :scale: 80 %

   @startuml
   class FooManager {
       +int    id
       +string name
       +init() : void
       +process(data : bytes) : bool
   }
   class BarService {
       -FooManager manager
       +execute() : void
   }
   FooManager <-- BarService : uses
   @enduml
```

#### State Machine
```rst
.. uml::
   :caption: Figure N — <Component> State Machine
   :scale: 80 %

   @startuml
   [*]        --> Idle
   Idle       --> Processing : startJob()
   Processing --> Done       : jobComplete()
   Processing --> Error      : jobFailed()
   Error      --> Idle       : reset()
   Done       --> [*]
   @enduml
```

#### ER Diagram
```rst
.. uml::
   :caption: Figure N — <Domain> Data Model
   :scale: 80 %

   @startuml
   entity "User" as USER {
       *id       : INTEGER <<PK>>
       --
       email     : VARCHAR(255)
       name      : VARCHAR(100)
   }
   entity "Order" as ORDER {
       *id       : INTEGER <<PK>>
       --
       *user_id  : INTEGER <<FK>>
       created_at : DATE
   }
   USER ||--o{ ORDER : places
   @enduml
```

#### Activity / Flowchart (for procedures)
```rst
.. uml::
   :caption: Figure N — <Procedure> Procedure Flowchart
   :scale: 80 %

   @startuml
   start
   :Precondition check;
   if (Precondition met?) then (yes)
       :Execute Step 1;
       :Execute Step 2;
   else (no)
       :Log failure;
       stop
   endif
   :Notify completion;
   stop
   @enduml
```

---

### 6. Build and Validate

```bash
cd docs
doxygen Doxyfile             # C/C++ only
make latexpdf 2>&1 | tee _build/build.log
grep -E "WARNING|ERROR" _build/build.log
# Verify PlantUML diagrams were rendered
ls _build/html/_images/*.png 2>/dev/null | head -5 || echo "WARNING: no diagram PNGs — check plantuml is on PATH"
```

If Sphinx emits `WARNING: plantuml command ... cannot be found`, run `which plantuml` to verify the binary is on PATH, then retry.

Resolve **all** Sphinx warnings before committing (broken references, missing files, etc.).

Verify the PDF at `docs/_build/latex/<project_abbr>.pdf`:
- Cover page: correct document number, version, date, safety class, confidentiality marking
- All sections present with real content (no "TBD")
- Revision history table populated
- Breathe/API sections are non-empty (for SDD)
- Page headers/footers show document number and classification

### 7. Update .gitignore

Ensure `docs/_build/` is excluded (the PDF is the deliverable, not the build artifacts):

```
docs/_build/
```

Add to `.gitignore` at the repo root if not already present.

### 8. Commit

```bash
git add docs/ .claude/doc-meta.yaml .gitignore docs/_plantuml_style.puml
git commit -m "docs(<doctype>): add/update <DOCTYPE> v<version>"
```

---

## Style Quick Reference

| Element | RST |
|---------|-----|
| Document title (H1) | `=` underline |
| Section (H2) | `-` underline |
| Subsection (H3) | `~` underline |
| Sub-subsection (H4) | `^` underline |
| Informational note | `.. note::` |
| Safety warning | `.. warning::` |
| Compliance requirement | `.. important::` |
| Internal cross-ref | `:ref:\`label\`` |
| Table | `.. list-table::` |
| C++ class API | `.. doxygenclass::` |
| C++ function API | `.. doxygenfunction::` |
| C struct | `.. doxygenstruct::` |
| Terminal command | `.. code-block:: bash` |
| PlantUML diagram | `.. uml::` |
| Diagram caption | `:caption: Figure N — Title` |

---

## Wrap Up

Provide the user with a summary in this format:

- **Documents created/updated**: list each file path
- **PDF location**: `docs/_build/latex/<name>.pdf`
- **Build result**: ✅ clean (0 warnings) or ‼️ warnings remaining (list them)
- **Document numbers**: list each DOC-XXXX-YYY-NNN assigned
- **Next steps**:
  - Upload the PDF to the DMS for formal review and approval
  - Before the next release, increment `version` and `release_date` in `.claude/doc-meta.yaml` and append a row to `revision_history.rst`
  - To make this skill available in all projects: `cp -r .claude/skills/doc-generation ~/.claude/skills/`
