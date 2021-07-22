# Security

## Contents

- [Kubernetes Security Primitives](#kubernetes-security-primitives)
- [Authentication](#authentication)
- [Setting up Basic Authentication](#setting-up-basic-authentication)
- [A note on Service Accounts](#a-note-on-service-accounts)
- [TLS](#tls)
  * [TLS Basics](#tls-basics)
  * [TLS in Kubernetes](#tls-in-kubernetes)
- [View Certificate Details](#view-certificate-details)
- [Certificates API](#certificates-api)
- [KubeConfig](#kubeconfig)
- [Persistent Key/Value Store](#persistent-key-value-store)
- [API Groups](#api-groups)
- [Authorization](#authorization)
- [Role Based Access Controls](#role-based-access-controls)
- [Cluster Roles and Role Bindings](#cluster-roles-and-role-bindings)
- [Image Security](#image-security)
- [Security Contexts](#security-contexts)
- [Network Policy](#network-policy)
- [Developing network policies](#developing-network-policies)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

## Kubernetes Security Primitives

### How to secure clusters?

#### Step 1 : Defining who can access ?

- Files - Username and Passwords
- Files - Username and Tokens
- Certificates
- External Authentication providers - LDAP
- Service Accounts

#### Step 2 : Once access is granted, defining what can they do ?

- RBAC Authorization
- ABAC Authorization
- Node Authorization
- Webhook Mode

## Authentication



## Setting up Basic Authentication

## A note on Service Accounts

## TLS 

### TLS Basics

### TLS in Kubernetes

## View Certificate Details

## Certificates API

## KubeConfig

## Persistent Key/Value Store

## API Groups

## Authorization

## Role Based Access Controls

## Cluster Roles and Role Bindings

## Image Security

## Security Contexts

## Network Policy

## Developing network policies



