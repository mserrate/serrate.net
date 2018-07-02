---
title: Introduction to MLflow
categories:
  - Machine Learning
tags:
  - BigData
  - ML
  - MachineLearning
thumbnail: /files/2018/07/1run_artifact.png
---

<a href="https://mlflow.org/" target="_blank">MLflow</a> is a tool to manage the lifecycle of Machine Learning projects. Is composed by three components:
* Tracking: Records parameters, metrics and artifacts of each run of a model
* Projects: Format for packaging data science projects and its dependencies
* Models: Generic format for packaging ML models and serve them through REST API or others.


## ML Tracking using XGBoost
Let's work on a quick sample to demonstrate the benefits of [MLFlow](https://www.mlflow.org) by tracking ML experiment using *XGBoost* and the [Census Income Data Set](https://archive.ics.uci.edu/ml/datasets/Census+Income).

{% codeblock lang:python %}
with mlflow.start_run():
    # model parameters
    params = {'learning_rate': 0.1, 'n_estimators': 100, 'seed': 0, 'subsample': 1, 'colsample_bytree': 1,
                  'objective': 'binary:logistic', 'max_depth': 3}

    # log model params
    for key in params:
        mlflow.log_param(key, params[key])

    # train XGBoost model
    gbtree = XGBClassifier(**params)
    gbtree.fit(train_features, train_labels)

    importances = gbtree.get_booster().get_fscore()
    print(importances)

    # get predictions
    y_pred = gbtree.predict(test_features)

    accuracy = accuracy_score(test_labels, y_pred)
    print("Accuracy: %.1f%%" % (accuracy * 100.0))

    # log accuracy metric
    mlflow.log_metric("accuracy", accuracy)

    sns.set(font_scale=1.5)
    xgb.plot_importance(gbtree)
    plt.savefig("importance.png", dpi = 200, bbox_inches = "tight")

    mlflow.log_artifact("importance.png")

    # log model
    mlflow.sklearn.log_model(gbtree, "model")
{% endcodeblock %}

In this example, we're using the MLflow Python API to track the experiment parameters, metric (accuracy), artifacts (our plot) and the XGBoost model.

When we run for the first time, we can see in the MLflow UI the following:

{% img /files/2018/07/1run.png %}
With our initial parameters we see that the metric accuracy is: **0.866 (86.6%)**

If we select the run and we see our artifact:

{% img /files/2018/07/1run_artifact.png %}

Next, we will change our parameter *max_depth* to 6 and let's see what happens:

{% img /files/2018/07/2run.png %}

And we see that our accuract has improved: **0.874 (87.4%)**

All the history is tracked, as well as the model itself, so it means we will have all our experiments history tracked and the performance on the model at one moment in time.

You can check the full sample in Github [https://github.com/mserrate/mlflow-sample](https://github.com/mserrate/mlflow-sample).