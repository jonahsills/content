---
title: 'On-device speech-to-text'
description: 'Speech recognition that runs locally on the user machine instead of sending audio to a hosted API.'
date: 2026-05-27
author: 'Jonah Sills'
---

# On-device speech-to-text

## Definition

On-device speech-to-text is automatic speech recognition that runs on the
same machine, workspace, or edge device that has access to the audio. Instead
of uploading recordings to a hosted transcription API, the application loads a
local model and produces text inside the local runtime.

## Context and Usage

Developers choose on-device speech-to-text when privacy, latency, offline use,
or predictable cost matters more than access to a managed cloud service. A
local model can be useful for voice commands, meeting-note prototypes, field
devices, and development workflows where audio should stay inside a controlled
workspace.

The tradeoff is operational ownership. Teams must install compatible model
packages, handle model downloads, and test performance on the machines that
will run transcription. Reproducible environments such as Daytona workspaces
help make that setup easier to repeat across contributors and projects.
