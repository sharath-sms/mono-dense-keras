Constrained Monotonic Neural Networks
================

<!-- WARNING: THIS FILE WAS AUTOGENERATED! DO NOT EDIT! -->

This Python library implements Monotonic Dense Layer as described in
Davor Runje, Sharath M. Shankaranarayana, “Constrained Monotonic Neural
Networks”, https://https://arxiv.org/abs/2205.11775.

If you use this library, please cite:

        @misc{https://doi.org/10.48550/arxiv.2205.11775,
          doi = {10.48550/ARXIV.2205.11775},
          url = {https://arxiv.org/abs/2205.11775},
          author = {Runje, Davor and Shankaranarayana, Sharath M.},
          title = {Constrained Monotonic Neural Networks},
          publisher = {arXiv},
          year = {2022},
          copyright = {Creative Commons Attribution Non Commercial Share Alike 4.0 International}
        }

## Install

``` sh
pip install mono_dense_keras
```

## License

The full text of the license is available at:

https://github.com/airtai/mono-dense-keras/blob/main/LICENSE

You are free to: - Share — copy and redistribute the material in any
medium or format

- Adapt — remix, transform, and build upon the material

The licensor cannot revoke these freedoms as long as you follow the
license terms.

Under the following terms: - Attribution — You must give appropriate
credit, provide a link to the license, and indicate if changes were
made. You may do so in any reasonable manner, but not in any way that
suggests the licensor endorses you or your use.

- NonCommercial — You may not use the material for commercial purposes.

- ShareAlike — If you remix, transform, or build upon the material, you
  must distribute your contributions under the same license as the
  original.

- No additional restrictions — You may not apply legal terms or
  technological measures that legally restrict others from doing
  anything the license permits.

## How to use

First, we’ll create a simple dataset for testing using numpy. Inputs
values $x_1$, $x_2$ and $x_3$ will be sampled from the normal
distribution, while the output value $y$ will be calculated according to
the following formula before adding noise to it:

$y = x_1^3 + \sin\left(\frac{x_2}{2 \pi}\right) + e^{-x_3}$

``` python
import numpy as np

rng = np.random.default_rng(42)

def generate_data(no_samples: int, noise: float):
    x = rng.normal(size=(no_samples, 3))
    y = x[:, 0] ** 3
    y += np.sin(x[:, 1] / (2*np.pi))
    y += np.exp(-x[:, 2])
    y += noise * rng.normal(size=no_samples)
    return x, y

x_train, y_train = generate_data(10_000, noise=0.1)
x_val, y_val = generate_data(10_000, noise=0.)
```

First, we’ll build a simple feedforward neural network using `Dense`
layer from Keras library.

``` python
import tensorflow as tf

from tensorflow.keras import Sequential
from tensorflow.keras.layers import Dense, Input
from tensorflow.keras.optimizers import Adam
from tensorflow.keras.optimizers.schedules import ExponentialDecay

# build a simple model with 3 hidden layer
model = Sequential()

model.add(Input(shape=(3,)))
model.add(Dense(128, activation="elu"))
model.add(Dense(128, activation="elu"))
model.add(Dense(1))
```

We’ll train the network using the `Adam` optimizer and the
`ExponentialDecay` learning rate schedule:

``` python
def train_model(model, initial_learning_rate):
    # train the model
    lr_schedule = ExponentialDecay(
        initial_learning_rate=initial_learning_rate,
        decay_steps=10_000,
        decay_rate=0.9,
    )
    optimizer = Adam(learning_rate=lr_schedule)
    model.compile(optimizer="adam", loss="mse")

    model.fit(x=x_train, y=y_train, batch_size=32, validation_data=(x_val, y_val), epochs=10)
    
train_model(model, initial_learning_rate=.1)
```

    Epoch 1/10
    313/313 [==============================] - 1s 2ms/step - loss: 9.1098 - val_loss: 9.2552
    Epoch 2/10
    313/313 [==============================] - 1s 2ms/step - loss: 7.7995 - val_loss: 8.3143
    Epoch 3/10
    313/313 [==============================] - 1s 2ms/step - loss: 7.5270 - val_loss: 8.0499
    Epoch 4/10
    313/313 [==============================] - 1s 2ms/step - loss: 7.2095 - val_loss: 7.5935
    Epoch 5/10
    313/313 [==============================] - 1s 2ms/step - loss: 6.0665 - val_loss: 6.7911
    Epoch 6/10
    313/313 [==============================] - 1s 2ms/step - loss: 3.1178 - val_loss: 1.5964
    Epoch 7/10
    313/313 [==============================] - 1s 2ms/step - loss: 1.1686 - val_loss: 0.8541
    Epoch 8/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.7370 - val_loss: 1.5969
    Epoch 9/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.6011 - val_loss: 0.3739
    Epoch 10/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.4458 - val_loss: 0.3114

Now, we’ll use the
[`MonotonicDense`](https://airtai.github.io/mono-dense-keras/monodenselayer.html#monotonicdense)
layer instead of `Dense` layer. By default, the
[`MonotonicDense`](https://airtai.github.io/mono-dense-keras/monodenselayer.html#monotonicdense)
layer assumes the output of the layer is monotonically increasing with
all inputs. This assumtion is always true for all layers except possibly
the first one. For the first layer, we use `indicator_vector` to specify
which input parameters are monotonic and to specify are they
increasingly or decreasingly monotonic: - set 1 for increasingly
monotonic parameter,

- set -1 for decreasingly monotonic parameter, and

- set 0 otherwise.

In our case, the `indicator_vector` is `[1, 0, -1]` because $y$ is: -
monotonically increasing w.r.t. $x_1$
$\left(\frac{\partial y}{x_1} = 3 {x_1}^2 \geq 0\right)$, and

- monotonically decreasing w.r.t. $x_3$
  $\left(\frac{\partial y}{x_3} = - e^{-x_2} \leq 0\right)$.

``` python
from airt.keras.layers import MonotonicDense


# build a simple model with 3 hidden layer, but this using MonotonicDense layer
mono_model = Sequential()

mono_model.add(Input(shape=(3,)))
indicator_vector = [1, 0, -1]
mono_model.add(MonotonicDense(128, activation="elu", indicator_vector=indicator_vector))
mono_model.add(MonotonicDense(128, activation="elu"))

mono_model.add(Dense(1))
```

``` python
train_model(model, initial_learning_rate=.001)
```

    Epoch 1/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.3646 - val_loss: 0.2042
    Epoch 2/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.2895 - val_loss: 0.1387
    Epoch 3/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.2756 - val_loss: 0.1027
    Epoch 4/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.2281 - val_loss: 0.0814
    Epoch 5/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.1816 - val_loss: 0.0634
    Epoch 6/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.1631 - val_loss: 0.1443
    Epoch 7/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.1455 - val_loss: 0.1299
    Epoch 8/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.1701 - val_loss: 0.0709
    Epoch 9/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.1250 - val_loss: 0.0644
    Epoch 10/10
    313/313 [==============================] - 1s 2ms/step - loss: 0.1426 - val_loss: 0.0405
