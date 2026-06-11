---
title: 自反思框架
tags:
  - self-improving
  - agent
publish: true
lang:
---
```python
import torch 
import torch.nn as nn
import torch.nn.functional as F





class SelfAttention(nn.Module):
	def __init__(self):
		self.embed_dim=embed_dim
		self.num_head=num_head
		self.head_dim=head_dim
		
		self.q_proj=nn.Linear(embed_dim,embed_dim)
		self.k_proj=nn.Linear(embed_dim,embed_dim)
		self.v_proj=nn.Linear(embed_dim,embed_dim)
		self.out_proj=nn.Linear(embed_dim,embed_dim)
		
		self.dropout=nn.Dropout(dropout)
		
		
	def attention
		






```