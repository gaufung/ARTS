---
date: 2017-04-26 22:04
status: public
title: 'Linear Classifier Implemetation'
---

Notes about learning [Standford CS231n](http://cs231n.github.io/) course
# 1 Linear classification
Linear clasifications are the foundamental components in machine learning theory. As before, the training dataset is $x_i \in R^{D}$, each associated with a label $y_i$. Here $i = 1\ldots N$ and $y_i \in 1\ldots K$. It means that we have $N$ examples with a dimensionality $D$ and $K$ distinct categories.   
The simplest possible function, a linear mapping
$$f(x_i, W,b)=Wx_i+b$$
where $W$ is the weights of function with $K\times D$ shape, $x_i$ is a column vector respresenting individual sample. And $b$ is bias which contains $K \times 1$ shape. So the $f$ is a column vector with $K$ elements which store the scores of each classe. To make the formula compact, we move $b$ into $W$. Here is the transformation.

![](~/com.png)
# 2 Loss function
We define as **Loss function** (or sometimes also refer to as the cost function or the objective) to measures out unhappiness if we misclassifier the categories.
## 2.1 Mulitclass SVM
The core concept of SVM is that correct class for each train exmaple is except to have the score higher than incorrect the uncorrect classes by a fixed margin $\Delta$. Let's present loss function the concept in mathematical formula:
$$L_i = \sum_{y\ne y_i}max(0, s_j-s_{y_i}+\Delta)$$
where $L_i$ is $i$th training example loss, $y_i$ is $i$th training example supposed classes. $s_j$ is the $i$th training example's scores in $j$th classes which is calculated by $ s_j=f(x_i,W)_j $. $y$ is individual class which represented in numerial numbers $0,\ldots,K-1$.

## 2.2 Softmax Classifier
Softmax classifier is another different loss fucntion which generates binary Logistic Regresssion classifier into multiple classes. Unlike the SVM which treats the outputs as the socres for each classes, the softmax classifier has   probabilistic interpretation. It replaces **hinge loss**  with a  **cross-entropy loss** that has this form:
$$L_i=-log(\frac{e^{f_{y_i}}}{\sum_je^{f_j}})$$
**Note:** To ensure the numeric stability, we need to make some pre-works before computation. If $f_j$ is big enough, the $e^{f_j}$ will be tremendous big that may cause the numeric unstability. We get the following mathematical equivalent expression:
$$\frac{e^{f_{y_i}}}{\sum_je^{f_j}} = \frac{C e^{f_{y_i}}}{C \sum_je^{f_j}}=\frac{e^{f_{y_i}+logC}}{\sum_je^{f_j+logC}}$$
where $logC = -max_jf_j$.
# 3 Regularization
To avoid overfitting,  we need to add penalty item into above loss functions which is called regualrization.  The common regualrization penalties are $L_1$ norm and $L_2$ norm. So the total loss fucntion as following:
$$L=\frac{1}{N}\sum_i L_i + \lambda R(W)$$
The first part is hinge loss or cross-entropy loss, the second part is regularization item. Our goal is to minimum the $L$ with optimized paramters in $W$
# 4 Gradient Descient
As we all know, the gradient is related to the direction which is function value's trend. If we calcualte the gradient of the obejctive function at particular point, we ought to know the next step to minimum. So we can update the parameters step by step.
# 5 Implementation
## 5.1 SVM classifier
```Python
def svm_loss_naive(W, X, y, reg, delta=1.0):
    '''
    Structured SVM loss function, navie implementation (with loops)
    Inputs:
    - W: C * D array where C is the no. of classes and D is feature's count
    - X: D * N array of data where D is feature's count and N is batch sample point size
    - y: 1-dimensional array of length N with labels 0...K-1 for K class
    - reg: (float) regularization strength
    - delta: the margin
    Returns: a tuple contianing
    - loss as single float
    - gradient with respect to weight W; an array of same shape of W
    '''
    dW = np.zeros(W.shape)
    # compute the loss and gradient
    num_classes = W.shape[0]
    num_train = X.shape[1]
    loss = 0.0
    for i in range(num_train):
        scores = W.dot(X[:, i])
        correct_class_score = scores[y[i]]
        for j in range(num_classes):
            if j == y[i]:
                continue
            margin = scores[j] - correct_class_score + delta
            if margin > 0:
                loss += margin
                # add gradient 
                dW[y[i], :] -= X[:, i].T
                dW[j, :] += X[:, i].T
    loss /= num_train
    dW /= num_train
    loss += 0.5 * reg * np.sum(W*W)
    dW += reg*W
    return loss, dW
```
## 5.2 Softmax classifier
```Python
def softmax_loss_naive(W, X, y, reg):
    '''
    Structured softmax loss function, naive implementation (with loops)
    Inputs:
    - W: C * D array where C is the no. of classes and D is feature's count
    - X: D * N array of data where D is feature's count and N is batch sample point size
    - y: 1-dimensional array of length N with labels 0...K-1 for K class
    - reg: (float) regularization strength
    Returns: a tuple contianing
    - loss as single float
    - gradient with respect to weight W; an array of same shape of W
    '''
    dW = np.zeros(W.shape)
    num_classes = W.shape[0]
    num_train = X.shape[1]
    loss = 0.0
    for i in range(num_train):
        scores = W.dot(X[:, i])
        dW[y[i], :] -= X[:, i].T
        #denominator = np.sum(np.exp(socres))
        logC = -1.0 * np.max(scores)
        denominator = np.sum(np.exp(scores + logC ))
        for j in range(num_classes):
            numerator = np.exp(scores[j] + logC)
            dW[j,:] += numerator/denominator * X[:,i].T
        loss += -1.0 * np.log(np.exp(scores[y[i]]+logC) / denominator)
    loss /= num_train
    dW /= num_train
    # regularization
    loss += 0.5 * reg * np.sum(W*W)
    dW += reg*W
    return loss, dW
```
# 6 Derivatives
Derivatives (scalar) and gradient (vector or matrix) are vital weapons to solve optimization problems. We assume that derivative is too simply to ignore it.
## 6.1 Vectors 
Support we have a column vector $\vec{y}$ with length $C$  calculated by forming the product between a matrx $W$ with $C$ rows and by $D$ columns and a vector $\vec{x}$ with $D$ length:
$$\vec{y}=W\vec{x}$$ 
Compute the derivatives of each component of $\vec{y}$ with respect to each component of $\vec{x}$, and we noted that there would be $C \times D$ of these.
$$\frac{\partial \vec{y}}{\partial \vec{x}} = W$$

## 6.2 Vector and Matrix
Let $\vec{y}$ be a row vector with $C$ components computed by taking the product of another row vector $\vec{x}$ with D components and a matrix $W$ that is $D$ rows by $C$ columns
$$\vec{y}=\vec{x}W$$
In general, when the index of the $\vec{y}$ component is equal to the second index of the $W$, the derivative will be non-zore, but will be zero otherwise, We can write:
$$\frac{\partial \vec{y_j}}{\partial W_{i,j}}=\vec{x_i}$$

## 6.3 Matrixs
Using multple examples of $\vec{x}$, stacked together to form a matrix $X$. Let us assume that each row represent individual $\vec{x}$ with length D. that X is a two-dimensional array with N rows and D columns, W, as in our last example, will be a matrix with D rows and C columns, Y will become N rows and C columns. Each row of Y will give a row vector associated with the corresponding row of the input X.
$$Y_{i,j}=\sum_{k=1}^{D}X_{i,k}W_{k,j}$$
if we let $Y_{i,:}$ be the ith row of $Y$ and let $X_{i,:}$ be the ith row of X, then we will see that $$\frac{\partial Y_{i,:}}{\partial X_{i,:}}=W$$