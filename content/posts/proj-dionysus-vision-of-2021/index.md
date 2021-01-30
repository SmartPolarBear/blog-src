---
title: "Project Dionysus : The Vision of 2021"
date: 2021-1-30T23:55:00+08:00
draft: false
slug: "project-dionysus-vision-of-2021"
description: "Setting out the vision for the grand image of future."
categories:
    - Blog
tags:
    - OS
---

Now, at the beginning of 2021, while celebrating the anniversary of this project, where most of my effort in the past year was devoted, it's vital to make clear the road head for it.
## Hindsight
What has been done for it in the past year and a few months? Above growing from the very first assembly file to a large project with tens of thousands of lines of code, these aspects stands out:

- **The Improvement of infrastructures of kernel** at the end of 2020, after 4 months struggling with file system, a minimum subset of EXT2 which support common operations like reading, creating and writing are implemented with a complete VFS abstraction. For the first time RAII and class hierarchy were adopted for kernel components. With the filesystem, all the modules of a usable kernel have been worked on more or less. 

- **Reliable build system has been adopted** Growing larger and larger, it's difficult to manage the project by traditional makefiles, which is why CMake was introduced at earlier time last year. And after the long time working on the project, the decision turns out to be right. CMake provides a powerful tool to manage the project, as well as external dependencies.

- **Code style and best practices were formed** Working long time with kernel, code styles like naming conventions have gained stability, and common best practices on kernel coding have been figured out.

## Now

- **Better process model**   
- **Optimized memory management ** 
- **Cache for file system ** 
- **Refactoring and resource leaks fixing ** 
## Future

- **Move things out of kernel** 

- **Improved synchronization infrastructures**

- **Rights and security**