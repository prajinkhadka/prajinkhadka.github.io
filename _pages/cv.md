---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

{% include base_path %}

Education
======
* Masters in Computer Science, [University of Nebraska Omaha]

Work experience
======
* Machine Learning Engineer, Fusemachines, 2019 - 2021
* Senior AI Engineer, Dogma Group, 2022 - 2023
* Google Summer Of Code, 2024
* Graduate Research Assistant, MC2 Lab, 2024 - Present

Skills
======
* Programming Languages: Python, R, C++, SQL, Java, Scala
* Machine Learning / Deep Learning: Transformer Models, LLMs (GPT, BERT), Graph Neural Networks (GNNs), Time-Series Forecasting
* Frameworks & Tools: PyTorch, TensorFlow, OpenCV, Hugging Face, LangChain, MLflow, FastAPI, Apache Airflow
* Data Processing: Apache Spark, Hadoop, Dask, Pandas, NumPy, Power BI
* Database & Deployment: AWS, GCP, Azure, MySQL, MongoDB, Redis, Docker, Faiss

Publications
======
  <ul>{% for post in site.publications %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>
  
Talks
======
  <ul>{% for post in site.talks %}
    {% include archive-single-talk-cv.html %}
  {% endfor %}</ul>
  
Teaching
======
  <ul>{% for post in site.teaching %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>
