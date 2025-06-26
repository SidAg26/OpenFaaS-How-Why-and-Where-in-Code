# OpenFaaS: How, Why, and Where in Code

This repository offers an AI-first, code-centric tutorial on OpenFaaS—a Kubernetes-native serverless framework. It's tailored for developers and enthusiasts keen on understanding the internal workings of OpenFaaS through practical code exploration.

---

## 📘 Overview

The tutorial is segmented into chapters, each delving into specific components of the OpenFaaS architecture. It emphasizes the "how," "why," and "where" of the codebase, facilitating a deeper comprehension of the system's design and functionality.

---

## 📂 Chapters

Each chapter dives into a specific building block of OpenFaaS and its Gateway implementation. The tutorial is structured to show not just what each component does, but how it does it in the codebase, and why it matters.

1. **[Gateway Application](Chapter_1.md)** — What is the Gateway? Why is it the control plane?
2. **[Configuration](Chapter_2.md)** — How is the Gateway configured? What are the knobs?
3. **[API Definition](Chapter_3.md)** — Where is the REST API defined and how does it route requests?
4. **[Request Middleware](Chapter_4.md)** — How does the Gateway transform and authenticate requests?
5. **[Request Handling (Sync/Async)](Chapter_5.md)** — How are functions invoked (sync/async)? Where does queueing happen?
6. **[Function Scaling](Chapter_6.md)** — How does OpenFaaS auto-scale functions from 0 to N?
7. **[Provider Interaction](Chapter_7.md)** — How does the Gateway interact with the Kubernetes provider?
8. **[Metrics & Monitoring](Chapter_8.md)** — How are Prometheus metrics exposed and what do they represent?

---

## 🚀 Getting Started

To make the most of this tutorial:

1. **Prerequisites**:

   * Basic understanding of Kubernetes and serverless concepts.
   * Familiarity with Go programming language.

2. **Clone the Repository**:

   ```bash
   git clone https://github.com/SidAg26/OpenFaaS-How-Why-and-Where-in-Code.git
   cd OpenFaaS-How-Why-and-Where-in-Code
   ```

3. **Navigate Through Chapters**:
   Each chapter is a standalone Markdown file. Start with `Overview.md` and proceed sequentially.

---

## 📄 License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

## 🙏 Credits

This tutorial was generated with the help of [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge), an open-source tool for producing structured educational walkthroughs from real-world codebases.

---

Feel free to customize this `README.md` further to align with any additional specifics or preferences you have for your repository.
