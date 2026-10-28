# Compiler

## Backend: Cranelift

The compiler is built on Cranelift.

## Optimizations — in the frontend

The explicit model gives precise aliasing, precise lifetimes, no hidden state → high-level optimizations (inlining, copy elimination, escape analysis) are done by the frontend.

Cranelift — code generation (registers, scheduling). We do not disable its passes — they are free and complementary.

## The main work: IR

The main engineering work is the IR (SSA, with the language's semantics):

```text
AST → IR → optimizations → Cranelift
```

## Flat AST (arena)

One `Vec<Node>`, indices instead of pointers — faster and simpler. Like an ECS in gamedev.

Completely without a tree — only a fast path for simple code; the general path is the arena (reference/box/parallelism analysis needs structure).

## Then the frontend decides

Inlining and move/copy (box) — the frontend decides by the number of uses.

## Backend-friendly IR

Study which patterns the Cranelift optimizer recognizes and give it exactly those.

## Syntax

The syntax does not change globally; micro-improvements for ease of parsing are possible.

## Parallel analysis

After the tree is built, it is analyzed across cores: each computation, action, or block is analyzed separately; interprocedural analysis is contract verification.

## SIMD prescan in the lexer

SIMD prescan in the lexer (like in Zoll).