---
layout: page
title: About
permalink: /about/
---

### Mingzhe Hu (胡明哲)

[Photo]({{site.url}}/figs/hmz.jpg)

### Contact

Email: humingzhework@163.com

### Research Interests

Programming Languages, Software Engineering, Security

### Education/Employment

- Ph.D. in Cyberspace Security, University of Science and Technology of China (USTC), 2020-?.
Advisor: Yan Xiong, Professor, and Yu Zhang, Associate Professor.
- M.S. in Cyberspace Security, University of Science and Technology of China (USTC), 2018-2020.
Advisor: Yu Zhang, Associate Professor.
- Software Engineer, WUXI CAS PHOTONICS Co., Ltd., 2016-2017.
- B.S. in Internet of Things, Jiangnan University, 2012-2016.

### Publications

- **Mingzhe Hu**, Yu Zhang, Wenchao Huang, and Yan Xiong. Static Type Inference for Foreign Functions of Python. 32nd International Symposium on Software Reliability Engineering (ISSRE 2021), pages 423-433, Wuhan, Hubei, China, October, 2021.
- Yun Peng, Yu Zhang, and **Mingzhe Hu**. An Empirical Study for Common Language
  Features Used in Python Projects. 28th IEEE International Conference on
  Software Analysis Evolution and Reengineering (SANER 2021), pages 24-35,
  virtual, March, 2021.
- **Mingzhe Hu**, and Yu Zhang. The Python/C API: Evolution, Usage Statistics, and Bug Patterns. 27th IEEE International Conference on Software Analysis Evolution and Reengineering (SANER 2020), pages 532-536, London, Ontario, Canada, Feb, 2020.

### Projects

#### PyCType

Static type inferencer for C foreign functions of Python.

[[Code](https://github.com/S4Plus/pyctype)], [[ISSRE Video](https://www.bilibili.com/video/BV1Kq4y1R7A9/)]

PyCType also checks the mismatch between the declaration and implementation of foreign functions, which makes a parameter-free foreign function can take arguments of any types.

Bugs found: [[NumPy-issue-18665](https://github.com/numpy/numpy/issues/18665)], [[Pillow-issue-5487](https://github.com/python-pillow/Pillow/issues/5487)].

#### PyCEAC

A static analysis toolset for extracting and analyzing the Python/C API from Python/C multilingual software systems (and CPython itself), bug-finding checkers for some related bug patterns as well.

[[Code](https://github.com/S4Plus/pyceac)], [Bugs found: mishandling exceptions ([Pillow-issue-3966](https://github.com/python-pillow/Pillow/issues/3966), [NumPy-issue-16773](https://github.com/numpy/numpy/issues/16773)), dynamic memory management flaws ([Pillow-issue-4115](https://github.com/python-pillow/Pillow/issues/4115))]
