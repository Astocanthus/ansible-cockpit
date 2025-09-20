# Ansible Cockpit

[![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white)](https://docs.ansible.com/ansible/latest/)
[![systemd](https://img.shields.io/badge/systemd-Quadlet-red?style=for-the-badge&logo=systemd&logoColor=white)](https://www.freedesktop.org/software/systemd/man/systemd.html)
[![Fedora CoreOS](https://img.shields.io/badge/Fedora%20CoreOS-Immutable%20OS-51A2DA?style=for-the-badge&logo=fedora&logoColor=white)](https://getfedora.org/coreos/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub release](https://img.shields.io/github/release/Astocanthus/ansible-cockpit.svg)](https://github.com/Astocanthus/ansible-cockpit/releases)
[![GitHub issues](https://img.shields.io/github/issues/Astocanthus/ansible-cockpit.svg)](https://github.com/Astocanthus/ansible-cockpit/issues)
![Status](https://img.shields.io/badge/Status-Development-red?logo=github)

> ⚠️ **Alpha Version**: This project is in active development and designed for distributed infrastructure environments. Features may change and should be tested thoroughly before production use.

A systemd-based ansible-pull orchestrator that provides state persistence and resume capability across reboots, specifically designed for distributed infrastructure deployments.

## Overview

Ansible Cockpit leverages **Quadlet** (systemd container management) and systemd services to create a robust ansible-pull execution environment that can:

- Execute ansible-pull playbooks automatically
- Track execution state and progress
- Resume playbook execution after system reboots
- Handle immutable OS scenarios (package installations requiring reboots)
- Provide logging and monitoring capabilities

## Architecture

```
┌─────────────────┐      ┌──────────────────┐     ┌───────────────────┐
│   Git Repository│      │  Ansible Cockpit │     │   Target System   │
│   (Playbooks)   │────▶│   Controller     │────▶│   (localhost)     │
└─────────────────┘      └──────────────────┘     └───────────────────┘
                              │
                              ▼
                       ┌──────────────────┐
                       │  State Database  │
                       │   (/var/lib/     │
                       │ ansible-cockpit) │
                       └──────────────────┘
```

## Key Features

### 🔄 State Persistence
- Tracks playbook execution progress in local SQLite database
- Resumes from last successful task after reboot
- Handles partial executions and failures

### 🐳 Quadlet Integration
- Uses systemd's Quadlet for container management
- Runs ansible-pull in isolated containers
- Automatic service restart and dependency management

### 📊 Progress Tracking
- Task-level progress tracking
- Execution logs with timestamps
- Status reporting (running, completed, failed, paused)

### 🔧 Reboot Handling
- Detects when reboots are required (e.g., kernel updates, system packages)
- Automatically resumes after system restart
- Maintains execution context across reboots

## High-Level Workflow

```
Start → Check Repository → Execute Playbook → Save State → Reboot? → Resume → Complete
  ↑                                             ↓                ↑          ↓
  └─────────────── System Restart ──────────────┘                └──────────┘
```

## Component Interaction

```
┌────────────────────────────────────────────────────────────────────┐
│                        systemd Services                            │
├────────────────────────────────────────────────────────────────────┤
│  ansible-cockpit.service                                           │
│  ├── Before boot completion                                        │
│  ├── Checks for pending execution                                  │
│  └── Launches ansible-pull with state tracking                     │
├────────────────────────────────────────────────────────────────────┤
│  ansible-cockpit-monitor.timer                                     │
│  ├── Periodic repository checks                                    │
│  └── Triggers execution on changes                                 │
└────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                      Quadlet Container                             │
├────────────────────────────────────────────────────────────────────┤
│  • Isolated ansible-pull execution environment                     │
│  • Mounts state directory from host                                │
│  • Network access for git repository                               │
│  • Manages ansible dependencies                                    │
└────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                    State Management                                │
├────────────────────────────────────────────────────────────────────┤
│  /var/lib/ansible-cockpit/                                         │
│  ├── state.json          (current execution state)                 │
│  ├── progress.log        (detailed execution log)                  │
│  ├── last_commit.txt     (repository version tracking)             │
│  └── execution_lock      (prevents concurrent runs)                │
└────────────────────────────────────────────────────────────────────┘
```

## State Transition Diagram

```
                     ┌─────────────┐
                     │   INITIAL   │
                     └──────┬──────┘
                           │
                           ▼
                     ┌─────────────┐
              ┌────▶│    RUNNING   │
              │      └──────┬──────┘     
              │            │            
              │            ▼            
    ┌─────────┴──┐  ┌──────────────┐    
    │  RESUMING  │  │REBOOT_PENDING│    
    └─────────┬──┘  └──────┬───────┘    
              │            │            
              │            ▼            
              │     ┌──────────────┐     
              └─────┤   REBOOTING  │
                    └──────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   COMPLETED  │
                    └──────────────┘
```

## Boot Sequence Integration

```
System Boot
     │
     ▼
┌─────────────────┐
│ Early Services  │
└─────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│             ansible-cockpit.service                     │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ 1. Check /var/lib/ansible-cockpit/state.json        │ │
│ │ 2. If REBOOT_PENDING → Resume execution             │ │
│ │ 3. If COMPLETED → Check for repository updates      │ │
│ │ 4. If INITIAL → Start fresh execution               │ │
│ └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────┐
│ Other Services  │
│ User Login      │
└─────────────────┘
```

## Immutable OS Scenario

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Package Installation Flow                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Start Playbook                                                         │
│       │                                                                 │
│       ▼                                                                 │
│  ┌─────────────────┐     ┌──────────────────┐                           │
│  │ Pre-reboot      │───▶│ Mark State:       │                          │
│  │ Tasks           │     │ "REBOOT_PENDING" │                           │
│  │ (repo clone,    │     │ Save progress    │                           │
│  │  preparations)  │     └──────────────────┘                           │
│  └─────────────────┘              │                                     │
│                                   ▼                                     │
│  ┌─────────────────┐     ┌──────────────────┐                           │
│  │ Layer Packages  │───▶│ Trigger Reboot    │                          │
│  │ (rpm-ostree)    │     │ systemctl reboot │                           │
│  └─────────────────┘     └──────────────────┘                           │
│                                   │                                     │
│                                   ▼                                     │
│                          ┌──────────────────┐                           │
│                          │  System Reboots  │                           │
│                          └──────────────────┘                           │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────┐     ┌──────────────────┐                           │
│  │ Boot Complete   │───▶│ ansible-cockpit   │                          │
│  │ New OS Layer    │     │ detects PENDING  │                           │
│  │ Active          │     │ state            │                           │
│  └─────────────────┘     └──────────────────┘                           │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────┐     ┌──────────────────┐                           │
│  │ Resume          │───▶│ Mark State:       │                          │
│  │ Post-reboot     │     │ "COMPLETED"      │                           │
│  │ Tasks           │     │                  │                           │
│  └─────────────────┘     └──────────────────┘                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```