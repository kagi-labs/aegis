# Project Aegis Architecture 🛡️

## Overview
Project Aegis is the Universal Security Control Plane. It lives on the **Client Side** as a MITM proxy for tool calls, enforcing safety and human-in-the-loop approvals.

## Human Interface
Aegis communicates with the local **Minato** instance to handle human approvals.
1. Aegis pauses tool execution.
2. Aegis sends an approval request to **Minato**.
3. Minato routes the request to the user's active channel.

## Storage & Trust
Aegis uses **Project Kura** to store its "Local Trust" records—verified versions of tools and skills are cached locally for speed and security.
