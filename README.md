# Ansible Dockers

## Goals

Setup multiple docker daemons on the same host, using Ansible

## Requirements

- Vagrant
- Ansible

## Setup

Spwan the vm :

    vagrant up

Test it :

    ssh root@ansible-dockers.example.com echo "Hello from $(hostname)"

If host is not found, run

    vagrant hostmanager

to update /etc/hosts

Then, make a snapshot to avoid rebuilding the all thing

    vagrant snapshot save base