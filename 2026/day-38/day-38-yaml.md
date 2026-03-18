# Day 38 – YAML Basics

## What is YAML?
YAML is a human-readable language used to write CI/CD pipelines, 
Kubernetes configs, Docker Compose files, and Ansible playbooks.

## Key Learnings

### 1. Indentation is everything
- Always use 2 spaces — never tabs
- Wrong indentation = error or wrong data

### 2. Types matter
- `true/false` = boolean
- `"true"` = string
- `42` = integer

### 3. Multi-line strings
- `|` = preserves newlines (use for scripts)
- `>` = folds into one line (use for descriptions)

## Two Ways to Write a List
# Block style
tools:
  - docker
  - kubernetes

# Inline style
hobbies: [reading, coding, gaming]

## Files Created
- person.yaml
- server.yaml
