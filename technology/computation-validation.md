# Computation Validation

To guarantee the integrity of geo-distributed GPU contributions, we introduce a computation validation mechanism. This mechanism uses two techniques.

* **Adaptive verification based on computation redundancy**. We replicate computations on multiple geo-distributed GPUs and compare their computation results. Such computation redundancy can consume extra AI compute resources. To minimize redundancy, we use random auditing: the redundancy is randomly created on random GPUs. Such a method creates an unpredictable verification system that intimidates dishonesty.&#x20;
* **Prediction-guided verification**. For the use cases of AI fine-tuning or training, we employ a continuous validation method. In particular, we analyze data patterns and the trend of model convergence during training. Then, we predict the model parameter evolution using empirical models \[1], and compare the prediction against the computation results. This dynamic validation mechanism can identify inconsistencies in parameter updates



Reference:

1. L. Liang, R. Chen, H. Chen, Y. Xia, K. Park, B. Zang, and H. Guan, “SLAQ: Quality-Driven Scheduling for Distributed Machine Learning,” in _ACM Symposium on Cloud Computing (SoCC)_, 2017
