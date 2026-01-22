# POISE Executor Agent - Architecture Specification

**Version:** 1.0  
**Generated:** 2026-01-22T19:12:53.297903  
**Session ID:** poise-arch-20260122-191253  
**Agent Type:** Executor (Resolution Plan Execution & Document Consolidation)

---

## Executive Summary

The **Executor Agent** receives resolution plans from the **Planner Agent** via the **A2A Protocol** (manual relay), executes multi-phase conflict resolution (semantic, style, link fixes), consolidates changes, and returns results. It operates in a stateless execution mode with comprehensive error handling and rollback capabilities.

---

## 1. Message Parser Module

### 1.1 A2A Message Parser

**Purpose:** Parse and validate incoming A2A protocol messages from Planner Agent.

**Implementation:**

```python
import json
import hashlib
from typing import Dict, Any, Optional
from dataclasses import dataclass

@dataclass
class A2AMessage:
    protocol_version: str
    message_type: str
    sender: str
    recipient: str
    session_id: str
    payload: Dict[str, Any]
    checksum: str

class A2AMessageParser:
    SUPPORTED_VERSIONS = ['1.0']
    SUPPORTED_MESSAGE_TYPES = ['resolution_request', 'verification_request']
    
    def parse(self, message_str: str) -> A2AMessage:
        try:
            data = json.loads(message_str)
        except json.JSONDecodeError as e:
            raise ValueError(f"Invalid JSON format: {e}")
        
        # Validate required fields
        required_fields = [
            'protocol_version', 'message_type', 'sender', 
            'recipient', 'session_id', 'payload', 'checksum'
        ]
        
        for field in required_fields:
            if field not in data:
                raise ValueError(f"Missing required field: {field}")
        
        # Validate protocol version
        if data['protocol_version'] not in self.SUPPORTED_VERSIONS:
            raise ValueError(f"Unsupported protocol version: {data['protocol_version']}")
        
        # Validate message type
        if data['message_type'] not in self.SUPPORTED_MESSAGE_TYPES:
            raise ValueError(f"Unsupported message type: {data['message_type']}")
        
        # Validate checksum
        if not self._verify_checksum(data):
            raise ValueError("Checksum verification failed")
        
        return A2AMessage(
            protocol_version=data['protocol_version'],
            message_type=data['message_type'],
            sender=data['sender'],
            recipient=data['recipient'],
            session_id=data['session_id'],
            payload=data['payload'],
            checksum=data['checksum']
        )
    
    def _verify_checksum(self, data: Dict[str, Any]) -> bool:
        payload_str = json.dumps(data['payload'], sort_keys=True)
        calculated_checksum = hashlib.sha256(payload_str.encode()).hexdigest()
        return calculated_checksum == data['checksum']
```


---

### 1.2 Resolution Plan Parser

**Purpose:** Parse the resolution plan payload from A2A messages.

```python
from typing import List
from enum import Enum

class ConflictType(Enum):
    SEMANTIC = 'semantic'
    STYLE = 'style'
    LINK = 'link'
    DEPENDENCY = 'dependency'

@dataclass
class ResolutionAction:
    action: str  # 'resolve', 'merge', 'replace'
    target: str  # file path
    details: Dict[str, Any]

@dataclass
class ResolutionPhase:
    type: ConflictType
    actions: List[ResolutionAction]
    priority: int = 0

@dataclass
class ResolutionPlan:
    plan_id: str
    phases: List[ResolutionPhase]
    estimated_time: int
    metadata: Dict[str, Any]

class ResolutionPlanParser:
    def parse(self, payload: Dict[str, Any]) -> ResolutionPlan:
        if 'plan_id' not in payload:
            raise ValueError("Missing plan_id in payload")
        
        phases = []
        for phase_data in payload.get('phases', []):
            actions = [
                ResolutionAction(
                    action=action_data['action'],
                    target=action_data['target'],
                    details=action_data['details']
                )
                for action_data in phase_data.get('actions', [])
            ]
            
            phases.append(ResolutionPhase(
                type=ConflictType(phase_data['type']),
                actions=actions,
                priority=phase_data.get('priority', 0)
            ))
        
        return ResolutionPlan(
            plan_id=payload['plan_id'],
            phases=sorted(phases, key=lambda p: p.priority, reverse=True),
            estimated_time=payload.get('estimated_time', 0),
            metadata=payload.get('metadata', {})
        )
```


---

## 2. Execution Engine

### 2.1 Phase Orchestrator

**Purpose:** Orchestrate multi-phase resolution execution with rollback support.

```python
from typing import Callable, List, Optional
import logging

logger = logging.getLogger(__name__)

@dataclass
class ExecutionResult:
    phase_type: ConflictType
    success: bool
    changes: List[Dict[str, Any]]
    errors: List[str]
    rollback_data: Optional[Dict[str, Any]] = None

class PhaseOrchestrator:
    def __init__(self):
        self.resolvers = {
            ConflictType.SEMANTIC: SemanticResolver(),
            ConflictType.STYLE: StyleResolver(),
            ConflictType.LINK: LinkResolver(),
            ConflictType.DEPENDENCY: DependencyResolver()
        }
        self.execution_history: List[ExecutionResult] = []
    
    def execute_plan(self, plan: ResolutionPlan) -> List[ExecutionResult]:
        results = []
        
        for phase in plan.phases:
            logger.info(f"Executing phase: {phase.type.value}")
            
            try:
                result = self._execute_phase(phase)
                results.append(result)
                self.execution_history.append(result)
                
                if not result.success:
                    logger.error(f"Phase {phase.type.value} failed. Initiating rollback.")
                    self._rollback_to_phase(len(results) - 1)
                    break
                    
            except Exception as e:
                logger.exception(f"Error executing phase {phase.type.value}")
                results.append(ExecutionResult(
                    phase_type=phase.type,
                    success=False,
                    changes=[],
                    errors=[str(e)]
                ))
                self._rollback_to_phase(len(results) - 1)
                break
        
        return results
    
    def _execute_phase(self, phase: ResolutionPhase) -> ExecutionResult:
        resolver = self.resolvers.get(phase.type)
        if not resolver:
            raise ValueError(f"No resolver found for {phase.type}")
        
        changes = []
        errors = []
        rollback_data = {}
        
        for action in phase.actions:
            try:
                change = resolver.resolve(action)
                changes.append(change)
                rollback_data[action.target] = change.get('original_content')
            except Exception as e:
                errors.append(f"Action failed for {action.target}: {str(e)}")
        
        return ExecutionResult(
            phase_type=phase.type,
            success=len(errors) == 0,
            changes=changes,
            errors=errors,
            rollback_data=rollback_data if rollback_data else None
        )
    
    def _rollback_to_phase(self, phase_index: int):
        for i in range(phase_index, -1, -1):
            result = self.execution_history[i]
            if result.rollback_data:
                logger.info(f"Rolling back phase: {result.phase_type.value}")
                self._restore_from_rollback(result.rollback_data)
    
    def _restore_from_rollback(self, rollback_data: Dict[str, Any]):
        from pathlib import Path
        
        for file_path, original_content in rollback_data.items():
            try:
                Path(file_path).write_text(original_content)
                logger.info(f"Restored: {file_path}")
            except Exception as e:
                logger.error(f"Failed to restore {file_path}: {e}")
```


---

### 2.2 Semantic Conflict Resolver

**Purpose:** Resolve semantic conflicts using LLM-based reconciliation.

```python
import spacy

class SemanticResolver:
    def __init__(self):
        self.nlp = spacy.load('en_core_web_trf')
    
    def resolve(self, action: ResolutionAction) -> Dict[str, Any]:
        details = action.details
        
        # Get the two conflicting statements
        text1 = details.get('text1', '')
        text2 = details.get('text2', '')
        file1 = details.get('file1', '')
        file2 = details.get('file2', '')
        
        # Read original content
        from pathlib import Path
        content1 = Path(file1).read_text()
        
        # Generate reconciled version
        reconciled = self._reconcile_statements(text1, text2)
        
        # Replace in document
        updated_content = content1.replace(text1, reconciled)
        
        # Write back
        Path(file1).write_text(updated_content)
        
        return {
            'action': 'semantic_resolution',
            'file': file1,
            'original_content': content1,
            'original_text': text1,
            'reconciled_text': reconciled,
            'change_type': 'replacement'
        }
    
    def _reconcile_statements(self, text1: str, text2: str) -> str:
        # Simplified reconciliation logic
        # In production, use LLM to generate reconciled statement
        
        doc1 = self.nlp(text1)
        doc2 = self.nlp(text2)
        
        # Check for negation in text2
        has_negation = any(token.dep_ == 'neg' for token in doc2)
        
        if has_negation:
            # text2 contradicts text1, prioritize more recent/specific
            return f"{text1} (Note: {text2})"
        else:
            # Merge both perspectives
            return f"{text1} {text2}"
```


---

### 2.3 Style Conflict Resolver

**Purpose:** Resolve style inconsistencies based on Vale rules.

```python
import subprocess
import re

class StyleResolver:
    def __init__(self, vale_config_path: str = '.vale.ini'):
        self.vale_config = vale_config_path
    
    def resolve(self, action: ResolutionAction) -> Dict[str, Any]:
        details = action.details
        file_path = details.get('file', '')
        line_number = details.get('line', 0)
        rule = details.get('rule', '')
        message = details.get('message', '')
        
        from pathlib import Path
        content = Path(file_path).read_text()
        lines = content.splitlines()
        
        # Apply style fix based on rule type
        if line_number > 0 and line_number <= len(lines):
            original_line = lines[line_number - 1]
            fixed_line = self._apply_style_fix(original_line, rule, message)
            lines[line_number - 1] = fixed_line
        
        updated_content = '\n'.join(lines)
        Path(file_path).write_text(updated_content)
        
        return {
            'action': 'style_resolution',
            'file': file_path,
            'original_content': content,
            'line': line_number,
            'rule': rule,
            'change_type': 'line_replacement'
        }
    
    def _apply_style_fix(self, line: str, rule: str, message: str) -> str:
        # Common style fixes
        if 'passive voice' in message.lower():
            # Simplified passive to active conversion
            return line  # In production, use NLP transformation
        
        elif 'heading' in message.lower() and 'capitalization' in message.lower():
            # Fix heading capitalization (title case)
            return line.title()
        
        elif 'trailing whitespace' in message.lower():
            return line.rstrip()
        
        elif 'oxford comma' in message.lower():
            # Add Oxford comma
            return re.sub(r',\s+(\w+)\s+and\s+', r', \1, and ', line)
        
        else:
            # Default: return original
            return line
```


---

### 2.4 Link Conflict Resolver

**Purpose:** Fix broken internal and external links.

```python
import requests
from pathlib import Path
from urllib.parse import urljoin

class LinkResolver:
    def __init__(self, base_url: str = None):
        self.base_url = base_url
    
    def resolve(self, action: ResolutionAction) -> Dict[str, Any]:
        details = action.details
        file_path = details.get('file', '')
        target = details.get('target', '')
        line_number = details.get('line', 0)
        link_type = 'internal' if not target.startswith('http') else 'external'
        
        content = Path(file_path).read_text()
        
        if link_type == 'internal':
            fixed_target = self._fix_internal_link(target, file_path)
        else:
            fixed_target = self._fix_external_link(target)
        
        # Replace link in content
        updated_content = content.replace(f']({target})', f']({fixed_target})')
        Path(file_path).write_text(updated_content)
        
        return {
            'action': 'link_resolution',
            'file': file_path,
            'original_content': content,
            'original_target': target,
            'fixed_target': fixed_target,
            'change_type': 'link_update'
        }
    
    def _fix_internal_link(self, link: str, current_file: str) -> str:
        # Try to resolve relative path
        current_dir = Path(current_file).parent
        
        # Common fixes
        fixes_to_try = [
            link,
            f"../{link}",
            f"../../{link}",
            link.replace('.md', '.html'),
            link.lower(),
        ]
        
        for fix in fixes_to_try:
            target_path = current_dir / fix
            if target_path.exists():
                return fix
        
        # If nothing works, return original with warning comment
        return f"{link} <!-- FIXME: Broken link -->"
    
    def _fix_external_link(self, url: str) -> str:
        # Try common fixes for external links
        fixes_to_try = [
            url,
            url.replace('http://', 'https://'),
            url.rstrip('/'),
            url + '/',
        ]
        
        for fix in fixes_to_try:
            try:
                response = requests.head(fix, timeout=5, allow_redirects=True)
                if response.status_code < 400:
                    return fix
            except:
                continue
        
        # If no fix works, return original
        return url
```


---

### 2.5 Dependency Conflict Resolver

**Purpose:** Resolve circular dependencies and orphaned documents.

```python
import networkx as nx

class DependencyResolver:
    def resolve(self, action: ResolutionAction) -> Dict[str, Any]:
        details = action.details
        conflict_message = details.get('message', '')
        
        if 'circular' in conflict_message.lower():
            return self._resolve_circular_dependency(details)
        elif 'orphaned' in conflict_message.lower():
            return self._resolve_orphaned_document(details)
        else:
            return {
                'action': 'dependency_resolution',
                'success': False,
                'error': 'Unknown dependency conflict type'
            }
    
    def _resolve_circular_dependency(self, details: Dict[str, Any]) -> Dict[str, Any]:
        files = details.get('files', [])
        
        # Strategy: Remove the weakest link in the cycle
        # In production, use more sophisticated analysis
        
        if len(files) >= 2:
            # Remove link from last file to first file
            file_to_modify = files[-1]
            target_to_remove = files
            
            from pathlib import Path
            content = Path(file_to_modify).read_text()
            
            # Find and remove links to target
            import re
            pattern = f'\[([^\]]+)\]\([^)]*{Path(target_to_remove).name}[^)]*\)'
            updated_content = re.sub(pattern, r'\1', content)
            
            Path(file_to_modify).write_text(updated_content)
            
            return {
                'action': 'circular_dependency_resolution',
                'file': file_to_modify,
                'original_content': content,
                'removed_link_to': target_to_remove,
                'change_type': 'link_removal'
            }
        
        return {'action': 'circular_dependency_resolution', 'success': False}
    
    def _resolve_orphaned_document(self, details: Dict[str, Any]) -> Dict[str, Any]:
        orphaned_file = details.get('file', '')
        
        # Strategy: Add link to index or README
        from pathlib import Path
        project_root = Path(orphaned_file).parent
        
        # Try to find index or README
        index_candidates = [
            project_root / 'README.md',
            project_root / 'index.md',
            project_root / 'INDEX.md'
        ]
        
        index_file = None
        for candidate in index_candidates:
            if candidate.exists():
                index_file = candidate
                break
        
        if index_file:
            content = index_file.read_text()
            link_text = Path(orphaned_file).stem.replace('-', ' ').title()
            new_link = f"- [{link_text}]({Path(orphaned_file).name})"
            
            # Add link to end of file
            updated_content = content + f"\n\n{new_link}"
            index_file.write_text(updated_content)
            
            return {
                'action': 'orphan_resolution',
                'file': str(index_file),
                'original_content': content,
                'added_link_to': orphaned_file,
                'change_type': 'link_addition'
            }
        
        return {'action': 'orphan_resolution', 'success': False}
```


---

## 3. Document Consolidator

### 3.1 Change Merger

**Purpose:** Consolidate all changes from execution phases.

```python
from typing import List, Dict, Any
from collections import defaultdict

class ChangeMerger:
    def merge_changes(self, execution_results: List[ExecutionResult]) -> Dict[str, Any]:
        # Group changes by file
        changes_by_file = defaultdict(list)
        
        for result in execution_results:
            if result.success:
                for change in result.changes:
                    file_path = change.get('file', '')
                    if file_path:
                        changes_by_file[file_path].append(change)
        
        # Consolidate changes
        consolidated = {
            'total_files_modified': len(changes_by_file),
            'files': {},
            'summary': {
                'semantic_changes': 0,
                'style_changes': 0,
                'link_changes': 0,
                'dependency_changes': 0
            }
        }
        
        for file_path, changes in changes_by_file.items():
            consolidated['files'][file_path] = {
                'change_count': len(changes),
                'changes': changes
            }
            
            # Count by type
            for change in changes:
                action_type = change.get('action', '')
                if 'semantic' in action_type:
                    consolidated['summary']['semantic_changes'] += 1
                elif 'style' in action_type:
                    consolidated['summary']['style_changes'] += 1
                elif 'link' in action_type:
                    consolidated['summary']['link_changes'] += 1
                elif 'dependency' in action_type:
                    consolidated['summary']['dependency_changes'] += 1
        
        return consolidated
```


---

### 3.2 Diff Generator

**Purpose:** Generate unified diffs for all changes.

```python
import difflib
from pathlib import Path

class DiffGenerator:
    def generate_diffs(self, changes: Dict[str, Any]) -> Dict[str, str]:
        diffs = {}
        
        for file_path, file_changes in changes.get('files', {}).items():
            original_content = None
            current_content = None
            
            # Get original from first change
            for change in file_changes['changes']:
                if 'original_content' in change:
                    original_content = change['original_content']
                    break
            
            # Get current content from file
            try:
                current_content = Path(file_path).read_text()
            except:
                continue
            
            if original_content and current_content:
                diff = difflib.unified_diff(
                    original_content.splitlines(keepends=True),
                    current_content.splitlines(keepends=True),
                    fromfile=f"a/{file_path}",
                    tofile=f"b/{file_path}"
                )
                diffs[file_path] = ''.join(diff)
        
        return diffs
```


---

## 4. Response Formatter

### 4.1 A2A Response Builder

**Purpose:** Format execution results as A2A protocol response.

```python
class A2AResponseFormatter:
    def format_response(
        self, 
        session_id: str, 
        execution_results: List[ExecutionResult],
        consolidated_changes: Dict[str, Any],
        diffs: Dict[str, str]
    ) -> str:
        success = all(r.success for r in execution_results)
        
        response = {
            'protocol_version': '1.0',
            'message_type': 'resolution_response',
            'sender': 'executor_agent',
            'recipient': 'planner_agent',
            'session_id': session_id,
            'payload': {
                'success': success,
                'execution_summary': {
                    'total_phases': len(execution_results),
                    'successful_phases': sum(1 for r in execution_results if r.success),
                    'failed_phases': sum(1 for r in execution_results if not r.success)
                },
                'changes': consolidated_changes,
                'diffs': diffs,
                'errors': [
                    error 
                    for result in execution_results 
                    for error in result.errors
                ]
            },
            'checksum': None
        }
        
        # Calculate checksum
        payload_str = json.dumps(response['payload'], sort_keys=True)
        response['checksum'] = hashlib.sha256(payload_str.encode()).hexdigest()
        
        return json.dumps(response, indent=2)
```


---

## 5. Performance Optimization

### 5.1 Parallel Execution

**Purpose:** Execute independent resolution actions in parallel.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import List

class ParallelExecutor:
    def __init__(self, max_workers: int = 4):
        self.max_workers = max_workers
    
    def execute_actions_parallel(
        self, 
        actions: List[ResolutionAction], 
        resolver
    ) -> List[Dict[str, Any]]:
        results = []
        
        with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            future_to_action = {
                executor.submit(resolver.resolve, action): action 
                for action in actions
            }
            
            for future in as_completed(future_to_action):
                action = future_to_action[future]
                try:
                    result = future.result()
                    results.append(result)
                except Exception as e:
                    results.append({
                        'action': action.action,
                        'target': action.target,
                        'success': False,
                        'error': str(e)
                    })
        
        return results
```


---

## 6. Error Handling

### 6.1 Retry Logic

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

class ExecutorAgent:
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=10),
        retry=retry_if_exception_type((ConnectionError, TimeoutError))
    )
    def execute_resolution_plan(self, a2a_message: str) -> str:
        # Parse message
        parser = A2AMessageParser()
        message = parser.parse(a2a_message)
        
        # Parse plan
        plan_parser = ResolutionPlanParser()
        plan = plan_parser.parse(message.payload)
        
        # Execute
        orchestrator = PhaseOrchestrator()
        results = orchestrator.execute_plan(plan)
        
        # Consolidate
        merger = ChangeMerger()
        changes = merger.merge_changes(results)
        
        # Generate diffs
        diff_gen = DiffGenerator()
        diffs = diff_gen.generate_diffs(changes)
        
        # Format response
        formatter = A2AResponseFormatter()
        response = formatter.format_response(
            message.session_id,
            results,
            changes,
            diffs
        )
        
        return response
```


---

## 7. Testing Strategy

### 7.1 Unit Tests

```python
import pytest

def test_a2a_message_parser():
    message = {
        'protocol_version': '1.0',
        'message_type': 'resolution_request',
        'sender': 'planner_agent',
        'recipient': 'executor_agent',
        'session_id': 'test-123',
        'payload': {'plan_id': 'plan-123', 'phases': []},
        'checksum': hashlib.sha256(
            json.dumps({'plan_id': 'plan-123', 'phases': []}, sort_keys=True).encode()
        ).hexdigest()
    }
    
    parser = A2AMessageParser()
    result = parser.parse(json.dumps(message))
    
    assert result.session_id == 'test-123'
    assert result.message_type == 'resolution_request'

def test_semantic_resolver():
    resolver = SemanticResolver()
    action = ResolutionAction(
        action='resolve',
        target='/tmp/test.md',
        details={
            'text1': 'The system is slow.',
            'text2': 'The system is not slow.',
            'file1': '/tmp/test.md',
            'file2': '/tmp/test2.md'
        }
    )
    
    # Create test file
    Path('/tmp/test.md').write_text('# Test\nThe system is slow.')
    
    result = resolver.resolve(action)
    
    assert result['action'] == 'semantic_resolution'
    assert 'original_content' in result
```


---

## 8. Deployment

### 8.1 Docker Configuration

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    git \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

RUN python -m spacy download en_core_web_trf

COPY . .

CMD ["python", "executor_agent.py"]
```


---

## 9. Performance Requirements

- **Message Parsing:** < 500ms
- **Single Phase Execution:** < 5 seconds
- **Document Consolidation:** < 2 seconds
- **Diff Generation:** < 1 second per file
- **Response Formatting:** < 500ms

---

## 10. Security Considerations

- Validate all A2A message checksums
- Sanitize file paths to prevent directory traversal
- Limit file size for processing (max 10MB per file)
- Rate limit external API calls
- Implement file backup before modifications

---

**End of Executor Agent Architecture Specification**

[^1]: poise-extended.md
[^2]: Universal-Development-Best-Practices-and-Data.md
[^3]: POISE-Agent-Research-Report.md
[^4]: A2A-Documentation-Agent-Architecture.md
[^5]: QUICK-REFERENCE-CARD.md
[^6]: EXECUTIVE-SUMMARY.md
[^7]: A2A-Documentation-Agent-Architecture.md
[^8]: POISE-Agent-Research-Report.md```