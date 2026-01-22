# POISE Planner Agent - Architecture Specification

**Version:** 1.0  
**Generated:** 2026-01-22T19:12:53.297903  
**Session ID:** poise-arch-20260122-191253  
**Agent Type:** Planner (Documentation Conflict Detection & Resolution Planning)

---

## Executive Summary

The **Planner Agent** is responsible for ingesting documentation files, detecting conflicts (semantic, style, link, dependency), generating resolution plans, and orchestrating multi-step workflows via **LangGraph**. It communicates with the **Executor Agent** through the **A2A Protocol** (manual relay via copy/paste) and persists state in **PostgreSQL**.

---

## 1. Ingestion Pipeline

### 1.1 File Parser Module

**Purpose:** Parse various documentation formats and extract structured content.

**Supported Formats:**
- Markdown (`.md`)
- reStructuredText (`.rst`)
- AsciiDoc (`.adoc`)

**Implementation:**

```python
from dataclasses import dataclass
from typing import List, Dict, Any
from pathlib import Path
import re

@dataclass
class DocumentNode:
    file_path: str
    format: str  # 'markdown', 'rst', 'asciidoc'
    title: str
    headings: List[Dict[str, Any]]
    links: List[Dict[str, Any]]
    raw_content: str
    metadata: Dict[str, Any]

class DocumentParser:
    def parse_file(self, file_path: Path) -> DocumentNode:
        suffix = file_path.suffix.lower()
        
        if suffix == '.md':
            return self._parse_markdown(file_path)
        elif suffix == '.rst':
            return self._parse_rst(file_path)
        elif suffix == '.adoc':
            return self._parse_asciidoc(file_path)
        else:
            raise ValueError(f"Unsupported format: {suffix}")
    
    def _parse_markdown(self, file_path: Path) -> DocumentNode:
        content = file_path.read_text(encoding='utf-8')
        
        # Extract headings
        heading_pattern = r'^(#{1,6})\s+(.+)$'
        headings = []
        for i, line in enumerate(content.splitlines(), 1):
            match = re.match(heading_pattern, line)
            if match:
                level = len(match.group(1))
                text = match.group(2).strip()
                headings.append({'level': level, 'text': text, 'line': i})
        
        # Extract links
        link_pattern = r'\[([^\]]+)\]\(([^)]+)\)'
        links = []
        for i, line in enumerate(content.splitlines(), 1):
            for match in re.finditer(link_pattern, line):
                links.append({
                    'type': 'internal' if match.group(2).startswith('#') else 'external',
                    'target': match.group(2),
                    'text': match.group(1),
                    'line': i
                })
        
        return DocumentNode(
            file_path=str(file_path),
            format='markdown',
            title=headings['text'] if headings else file_path.stem,
            headings=headings,
            links=links,
            raw_content=content,
            metadata={'line_count': len(content.splitlines())}
        )
```


---

### 1.2 Metadata Extraction

**Purpose:** Extract project-specific metadata and conventions.

```python
import json

class MetadataExtractor:
    def extract_project_metadata(self, project_root: Path) -> Dict[str, Any]:
        metadata = {
            'project_name': None,
            'version': None,
            'style_guide': None,
            'link_base_url': None,
        }
        
        # Check package.json
        package_json = project_root / 'package.json'
        if package_json.exists():
            data = json.loads(package_json.read_text())
            metadata['project_name'] = data.get('name')
            metadata['version'] = data.get('version')
        
        # Check for Vale config
        vale_config = project_root / '.vale.ini'
        if vale_config.exists():
            metadata['style_guide'] = 'vale'
        
        return metadata
```


---

### 1.3 Structure Analysis

**Purpose:** Build a documentation structure graph for dependency analysis.

```python
import networkx as nx

class StructureAnalyzer:
    def build_dependency_graph(self, documents: List[DocumentNode]) -> nx.DiGraph:
        graph = nx.DiGraph()
        
        # Add nodes
        for doc in documents:
            graph.add_node(doc.file_path, title=doc.title, format=doc.format)
        
        # Add edges based on links
        for doc in documents:
            for link in doc.links:
                if link['type'] == 'internal':
                    target = self._resolve_internal_link(link['target'], doc.file_path)
                    if target in graph:
                        graph.add_edge(doc.file_path, target, link_text=link['text'])
        
        return graph
    
    def _resolve_internal_link(self, link: str, current_file: str) -> str:
        if link.startswith('../'):
            return str(Path(current_file).parent.parent / link[3:])
        return str(Path(current_file).parent / link)
```


---

## 2. Conflict Detection Engine

### 2.1 Semantic Conflict Detector

**Purpose:** Detect contradictory statements using NLP.

**Dependencies:** `spacy` (en_core_web_trf model)

```python
import spacy
from typing import List, Tuple

class SemanticConflictDetector:
    def __init__(self):
        self.nlp = spacy.load('en_core_web_trf')
    
    def detect_conflicts(self, documents: List[DocumentNode]) -> List[Dict[str, Any]]:
        conflicts = []
        
        # Extract sentences from all documents
        sentences = []
        for doc in documents:
            doc_nlp = self.nlp(doc.raw_content)
            for sent in doc_nlp.sents:
                sentences.append({
                    'text': sent.text,
                    'file': doc.file_path,
                    'embedding': sent.vector
                })
        
        # Compare sentence pairs for contradictions
        for i, sent1 in enumerate(sentences):
            for sent2 in sentences[i+1:]:
                if self._are_contradictory(sent1, sent2):
                    conflicts.append({
                        'type': 'semantic',
                        'severity': 'high',
                        'file1': sent1['file'],
                        'file2': sent2['file'],
                        'text1': sent1['text'],
                        'text2': sent2['text'],
                        'confidence': 0.85
                    })
        
        return conflicts
    
    def _are_contradictory(self, sent1: Dict, sent2: Dict) -> bool:
        import numpy as np
        
        similarity = np.dot(sent1['embedding'], sent2['embedding']) / (
            np.linalg.norm(sent1['embedding']) * np.linalg.norm(sent2['embedding'])
        )
        
        negation_words = {'not', 'no', 'never', 'neither', 'cannot', "won't"}
        has_negation = any(word in sent2['text'].lower() for word in negation_words)
        
        return similarity > 0.7 and has_negation
```


---

### 2.2 Style Conflict Detector

**Purpose:** Detect style inconsistencies using Vale.

```python
import subprocess
from pathlib import Path

class StyleConflictDetector:
    def __init__(self, vale_config: Path):
        self.vale_config = vale_config
    
    def detect_conflicts(self, documents: List[DocumentNode]) -> List[Dict[str, Any]]:
        conflicts = []
        
        for doc in documents:
            result = subprocess.run(
                ['vale', '--config', str(self.vale_config), '--output=JSON', doc.file_path],
                capture_output=True,
                text=True
            )
            
            if result.returncode != 0:
                vale_output = json.loads(result.stdout)
                for file_path, issues in vale_output.items():
                    for issue in issues:
                        conflicts.append({
                            'type': 'style',
                            'severity': issue['Severity'],
                            'file': file_path,
                            'line': issue['Line'],
                            'message': issue['Message'],
                            'rule': issue['Check']
                        })
        
        return conflicts
```


---

### 2.3 Link Conflict Detector

**Purpose:** Detect broken or inconsistent links.

```python
import requests

class LinkConflictDetector:
    def detect_conflicts(self, documents: List[DocumentNode], base_url: str = None) -> List[Dict[str, Any]]:
        conflicts = []
        
        for doc in documents:
            for link in doc.links:
                if link['type'] == 'external':
                    if not self._is_valid_external_link(link['target']):
                        conflicts.append({
                            'type': 'link',
                            'severity': 'medium',
                            'file': doc.file_path,
                            'line': link['line'],
                            'target': link['target'],
                            'message': f"Broken external link: {link['target']}"
                        })
                else:
                    if not self._is_valid_internal_link(link['target'], doc.file_path):
                        conflicts.append({
                            'type': 'link',
                            'severity': 'high',
                            'file': doc.file_path,
                            'line': link['line'],
                            'target': link['target'],
                            'message': f"Broken internal link: {link['target']}"
                        })
        
        return conflicts
    
    def _is_valid_external_link(self, url: str) -> bool:
        try:
            response = requests.head(url, timeout=5, allow_redirects=True)
            return response.status_code < 400
        except:
            return False
    
    def _is_valid_internal_link(self, link: str, current_file: str) -> bool:
        if link.startswith('#'):
            return True
        else:
            target_path = Path(current_file).parent / link
            return target_path.exists()
```


---

### 2.4 Dependency Conflict Detector

**Purpose:** Detect circular or missing dependencies.

```python
class DependencyConflictDetector:
    def detect_conflicts(self, graph: nx.DiGraph) -> List[Dict[str, Any]]:
        conflicts = []
        
        # Detect circular dependencies
        try:
            cycles = list(nx.simple_cycles(graph))
            for cycle in cycles:
                conflicts.append({
                    'type': 'dependency',
                    'severity': 'high',
                    'message': 'Circular dependency detected',
                    'files': cycle
                })
        except:
            pass
        
        # Detect orphaned documents
        for node in graph.nodes():
            if graph.in_degree(node) == 0 and graph.out_degree(node) == 0:
                conflicts.append({
                    'type': 'dependency',
                    'severity': 'low',
                    'message': 'Orphaned document (no incoming/outgoing links)',
                    'file': node
                })
        
        return conflicts
```


---

## 3. LangGraph Orchestrator

### 3.1 State Machine Definition

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class PlannerState(TypedDict):
    session_id: str
    documents: List[DocumentNode]
    conflicts: Annotated[List[Dict[str, Any]], operator.add]
    dependency_graph: Optional[nx.DiGraph]
    resolution_plan: Optional[Dict[str, Any]]
    a2a_message: Optional[str]
    status: str

def create_planner_workflow() -> StateGraph:
    workflow = StateGraph(PlannerState)
    
    # Add nodes
    workflow.add_node("ingest", ingest_documents)
    workflow.add_node("detect_semantic", detect_semantic_conflicts)
    workflow.add_node("detect_style", detect_style_conflicts)
    workflow.add_node("detect_links", detect_link_conflicts)
    workflow.add_node("detect_dependencies", detect_dependency_conflicts)
    workflow.add_node("generate_plan", generate_resolution_plan)
    workflow.add_node("format_a2a", format_a2a_message)
    
    # Add edges
    workflow.set_entry_point("ingest")
    workflow.add_edge("ingest", "detect_semantic")
    workflow.add_edge("detect_semantic", "detect_style")
    workflow.add_edge("detect_style", "detect_links")
    workflow.add_edge("detect_links", "detect_dependencies")
    workflow.add_edge("detect_dependencies", "generate_plan")
    workflow.add_edge("generate_plan", "format_a2a")
    workflow.add_edge("format_a2a", END)
    
    return workflow.compile()
```


---

### 3.2 Node Implementations

```python
def ingest_documents(state: PlannerState) -> PlannerState:
    parser = DocumentParser()
    state['status'] = 'detecting'
    return state

def detect_semantic_conflicts(state: PlannerState) -> PlannerState:
    detector = SemanticConflictDetector()
    conflicts = detector.detect_conflicts(state['documents'])
    state['conflicts'].extend(conflicts)
    return state

def generate_resolution_plan(state: PlannerState) -> PlannerState:
    plan = {
        'plan_id': f"plan-{state['session_id']}",
        'phases': [],
        'estimated_time': 0
    }
    
    for conflict in state['conflicts']:
        phase = {
            'type': conflict['type'],
            'actions': [{
                'action': 'resolve',
                'target': conflict.get('file', conflict.get('file1')),
                'details': conflict
            }]
        }
        plan['phases'].append(phase)
    
    state['resolution_plan'] = plan
    state['status'] = 'messaging'
    return state
```


---

## 4. State Persistence Layer

### 4.1 PostgreSQL Schema

```sql
CREATE TABLE planner_sessions (
    session_id VARCHAR(64) PRIMARY KEY,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    status VARCHAR(32) NOT NULL,
    metadata JSONB
);

CREATE TABLE planner_documents (
    id SERIAL PRIMARY KEY,
    session_id VARCHAR(64) REFERENCES planner_sessions(session_id),
    file_path TEXT NOT NULL,
    format VARCHAR(32) NOT NULL,
    title TEXT,
    raw_content TEXT,
    parsed_data JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE planner_conflicts (
    id SERIAL PRIMARY KEY,
    session_id VARCHAR(64) REFERENCES planner_sessions(session_id),
    conflict_type VARCHAR(32) NOT NULL,
    severity VARCHAR(16) NOT NULL,
    details JSONB NOT NULL,
    resolved BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE planner_resolution_plans (
    plan_id VARCHAR(64) PRIMARY KEY,
    session_id VARCHAR(64) REFERENCES planner_sessions(session_id),
    plan_data JSONB NOT NULL,
    a2a_message TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```


---

### 4.2 Repository Pattern

```python
from sqlalchemy import create_engine, Column, String, Integer, Text, Boolean, TIMESTAMP, JSON
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

class PlannerSessionModel(Base):
    __tablename__ = 'planner_sessions'
    session_id = Column(String(64), primary_key=True)
    status = Column(String(32), nullable=False)
    metadata = Column(JSON)

class PlannerRepository:
    def __init__(self, db_url: str):
        self.engine = create_engine(db_url)
        self.Session = sessionmaker(bind=self.engine)
    
    def save_session(self, session: PlannerState):
        db = self.Session()
        model = PlannerSessionModel(
            session_id=session['session_id'],
            status=session['status'],
            metadata={'conflict_count': len(session['conflicts'])}
        )
        db.merge(model)
        db.commit()
        db.close()
```


---

## 5. A2A Message Formatter

### 5.1 Message Structure

```python
import hashlib
import json

class A2AMessageFormatter:
    def format_resolution_request(self, plan: Dict[str, Any], session_id: str) -> str:
        message = {
            'protocol_version': '1.0',
            'message_type': 'resolution_request',
            'sender': 'planner_agent',
            'recipient': 'executor_agent',
            'session_id': session_id,
            'payload': plan,
            'checksum': None
        }
        
        payload_str = json.dumps(message['payload'], sort_keys=True)
        message['checksum'] = hashlib.sha256(payload_str.encode()).hexdigest()
        
        return json.dumps(message, indent=2)
```


---

## 6. Error Handling \& Retry Logic

```python
from tenacity import retry, stop_after_attempt, wait_exponential

class PlannerAgent:
    @retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
    def process_documents(self, file_paths: List[Path]) -> PlannerState:
        try:
            workflow = create_planner_workflow()
            initial_state = {
                'session_id': generate_session_id(),
                'documents': [],
                'conflicts': [],
                'dependency_graph': None,
                'resolution_plan': None,
                'a2a_message': None,
                'status': 'ingesting'
            }
            
            result = workflow.invoke(initial_state)
            return result
        except Exception as e:
            print(f"Error processing documents: {e}")
            raise
```


---

## 7. Testing Strategy

### 7.1 Unit Tests

```python
import pytest

def test_markdown_parser():
    parser = DocumentParser()
    content = "# Title\n\n## Heading\n\nSome text [link](target.md)"
    
    test_file = Path('/tmp/test.md')
    test_file.write_text(content)
    
    doc = parser.parse_file(test_file)
    
    assert doc.title == "Title"
    assert len(doc.headings) == 2
    assert len(doc.links) == 1
    assert doc.links['target'] == 'target.md'
```


---

## 8. Deployment

### 8.1 Docker Configuration

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y vale && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

RUN python -m spacy download en_core_web_trf

COPY . .

CMD ["python", "planner_agent.py"]
```


---

## 9. Performance Requirements

- **Document Parsing:** < 2 seconds per document
- **Conflict Detection:** < 10 seconds for 100 documents
- **Resolution Planning:** < 5 seconds
- **A2A Message Formatting:** < 1 second

---

## 10. Security Considerations

- Sanitize all file inputs
- Validate A2A message checksums
- Use prepared statements for SQL queries
- Rate limit external link checking

---

**End of Planner Agent Architecture Specification**

[^1]: poise-extended.md
[^2]: Universal-Development-Best-Practices-and-Data.md
[^3]: POISE-Agent-Research-Report.md
[^4]: A2A-Documentation-Agent-Architecture.md
[^5]: QUICK-REFERENCE-CARD.md
[^6]: EXECUTIVE-SUMMARY.md
[^7]: A2A-Documentation-Agent-Architecture.md
[^8]: POISE-Agent-Research-Report.md```