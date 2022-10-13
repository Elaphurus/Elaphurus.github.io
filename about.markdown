---
layout: page
title: About
permalink: /about/
---

### Mingzhe Hu (胡明哲)

[Photo]({{site.url}}/figs/hmz.jpg)

### Contact

EMail: humingzhework@163.com

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

- **Mingzhe Hu**, Yu Zhang. <u>An Empirical Study of the Python/C API on Evolution and Bug Patterns</u>. Journal of Software: Evolution and Process. 2022; e2507. <https://doi.org/10.1002/smr.2507>
- **Mingzhe Hu**, Yu Zhang, Wenchao Huang, and Yan Xiong. <u>Static Type Inference for Foreign Functions of Python</u>. 32nd International Symposium on Software Reliability Engineering (ISSRE 2021), pages 423-433, Wuhan, Hubei, China, October, 2021. <https://doi.org/10.1109/ISSRE52982.2021.00051>
- Yun Peng, Yu Zhang, and **Mingzhe Hu**. <u>An Empirical Study for Common Language Features Used in Python Projects</u>. 28th IEEE International Conference on Software Analysis Evolution and Reengineering (SANER 2021), pages 24-35, virtual, March, 2021. <https://doi.org/10.1109/SANER50967.2021.00012>
- **Mingzhe Hu**, and Yu Zhang. <u>The Python/C API: Evolution, Usage Statistics, and Bug Patterns</u>. 27th IEEE International Conference on Software Analysis Evolution and Reengineering (SANER 2020 ERA), pages 532-536, London, Ontario, Canada, Feb, 2020. <https://doi.org/10.1109/SANER48275.2020.9054835>

- <u>一种 Python 外部函数的静态类型推断方法及系统</u>. 发明人: 张昱, **胡明哲**. 申请号: CN202111105813.3A. 申请日: 20210922. 公开号: CN113885854A. 公开日: 20220104. <https://patents.google.com/patent/CN113885854A>
- <u>一种 Python 语言特征自动识别系统和方法</u>. 申请人: 中国科学技术大学, 发明人: 张昱, 彭昀, **胡明哲**. 专利号: ZL202010663123.9, 申请日: 20200710, 公开日: 20201030, 授权公告日: 20220111, 公开号: CN111858322B, 证书号: 4889779. <https://patents.google.com/patent/CN111858322B>

- Robert Harper 著, 张昱, **胡明哲**等译. <u>实用编程语言理论基础</u>. ISBN: 978-7-111-69740-4. 机械工业出版社. <http://www.hzcourse.com/web/teachRes/detail/5352/208>

### Projects

#### PyCType

Static type inferencer for C foreign functions of Python.

[[Code](https://github.com/S4Plus/pyctype)], [[ISSRE Video](https://www.bilibili.com/video/BV1Kq4y1R7A9/)]

PyCType also checks the mismatch between the declaration and implementation of foreign functions, which makes a parameter-free foreign function can take arguments of any types.

Bugs found: [[NumPy-issue-18665](https://github.com/numpy/numpy/issues/18665)], [[Pillow-issue-5487](https://github.com/python-pillow/Pillow/issues/5487)].

#### PyCEAC

A static analysis toolset for extracting and analyzing the Python/C API from Python/C multilingual software systems (and CPython itself), bug-finding checkers for some related bug patterns as well.

[[Code](https://github.com/S4Plus/pyceac)], [Bugs found: mishandling exceptions ([Pillow-issue-3966](https://github.com/python-pillow/Pillow/issues/3966), [NumPy-issue-16773](https://github.com/numpy/numpy/issues/16773)), dynamic memory management flaws ([Pillow-issue-4115](https://github.com/python-pillow/Pillow/issues/4115))]
