
# Machine Learning & linear regression


## 安装环境

```

brew install pyenv

pyenv install 3.7.3

pyenv global 3.7.3

pip install virtualenv
```

## 建立工作目录

```
mkdir coursera

cd coursera

virtualenv venv

source venv/bin/activate
```

## 安装 python lib
```
pip install notebook

pip install -U turicreate

pip install -U  matplotlib
```
启动 notebook
```
jupyter notebook
```

## turicreate not found issue
```
python -m pip install turicreate
python -m jupyter notebook
```


## Model
```
x - feature, covariate, predictor

y - observation, response



fw(x) = w0 + w1*X

w0: intercept   
w1: slope

w=(w0, w1)
```

## Residual sum of squares (RSS)     
```
所有数据点 离预测线的距离之平方和

RSS(w0, w1) = ($house1 - [w0 + w1*sq.fthouse1])2 + ($house2 - [w0 + w1*sq.fthouse2])2  +  .......   
为什么要平方？

对不同的 w0, w1
RSS(w0, w1) 最小的就是最准的模型

linear  线性

quadratic  二次
fw(x) = w0 + w1*X + w2*x*x

```


## Simulate prediction
```
不是RSS(w0, w1) 越小越好， 对老数据误差很小，但对新数据的预期可能很差。


把一部分数据放入 training error data set， 得出最小RSS， 在用这个RSS 去预期 其他剩下的数据 test error。
```


add more feature

fw(x) = w0 + w1*X + w2*y

## API doc
```
https://apple.github.io/turicreate/docs/api/turicreate.toolkits.regression.html#linear-regression


https://apple.github.io/turicreate/docs/api/generated/turicreate.SFrame.html#turicreate.SFrame
```


## code

```
// load lib
import turicreate   

// load data
sales = turicreate.SFrame('~/data/home_data.sframe/') 

// print data
sales

// Materializing X axis SArray
// Materializing Y axis SArray
turicreate.show(sales[1:5000]['sqft_living'],sales[1:5000]['price'])


// split data into training_error, test_error
training_set, test_set = sales.random_split(.8,seed=0)


// train simple regression model
sqft_model = turicreate.linear_regression.create(training_set,target='price',features=['sqft_living'])

// print model parameter
sqft_model.coefficients

// Evaluate the quality of our model
print (sqft_model.evaluate(test_set))
//  {'max_error': 4124342.9973428757, 'rmse': 255253.51816100787}


// draw predict chart
import matplotlib.pyplot as plt
%matplotlib inline
plt.plot(test_set['sqft_living'],test_set['price'],'.',
        test_set['sqft_living'],sqft_model.predict(test_set),'-')



// add more features
my_features = ['bedrooms','bathrooms','sqft_living','sqft_lot','floors','zipcode']
sales[my_features].show()

// build model
my_features_model = turicreate.linear_regression.create(training_set,target='price',features=my_features)

// Compare simple model with more complex one
print (sqft_model.evaluate(test_set))
print (my_features_model.evaluate(test_set))



//  predict
print (house1['price'])
print (sqft_model.predict(house1))
print (my_features_model.predict(house1))





```
