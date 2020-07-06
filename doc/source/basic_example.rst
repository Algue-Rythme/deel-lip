Example and usage
=================


In order to make things simple the following rules have been followed during development:

* ``deel-lip`` follows the ``keras`` package structure.
* All elements (layers, activations, initializers, ...) are compatible with standard the ``keras`` elements.
* When a k-Lipschitz layer overrides a standard keras layer, it uses the same interface and the same parameters.
  The only difference is a new parameter to control the Lipschitz constant of a layer.

Which layers are safe to use ?
------------------------------

The following table indicate which layer is safe to use in a lipshitz network, and which one is not.

.. role:: raw-html-m2r(raw)
   :format: html


.. list-table::
   :header-rows: 1

   * - layer
     - 1-lip?
     - deel-lip equivalent
     - comments
   * - Dense
     - no
     - SpectralDense\ :raw-html-m2r:`<br>`\ FrobeniusDense
     - SpectralDense and :raw-html-m2r:`<br>`\ FrobeniusDense are similar\ :raw-html-m2r:`<br>`\ when there is a single :raw-html-m2r:`<br>`\ output
   * - Conv2D
     - no
     - SpectralConv2D\ :raw-html-m2r:`<br>`\ FrobeniusConv2D
     - SpectralConv2D also :raw-html-m2r:`<br>`\ implement Björck :raw-html-m2r:`<br>`\ normalization
   * - MaxPooling\ :raw-html-m2r:`<br>`\ GlobalMaxPooling
     - yes
     - na.
     -
   * - AveragePooling\ :raw-html-m2r:`<br>`\ GlobalAveragePooling
     - no
     - ScaledAveragePooling\ :raw-html-m2r:`<br>`\ ScaledGlobalAveragePooling
     - The lipschitz constant :raw-html-m2r:`<br>`\ is bounded by :raw-html-m2r:`<br>`\ sqrt(pool_h*pool_h)
   * - Flatten
     - yes
     - na.
     -
   * - Dropout
     - no
     - None
     - The lipschitz constant :raw-html-m2r:`<br>`\ is bounded by the :raw-html-m2r:`<br>`\ dropout factor
   * - BatchNorm
     - no
     - None
     - It is suspected that :raw-html-m2r:`<br>`\ layer normalization already\ :raw-html-m2r:`<br>`\ limits internal covariate\ :raw-html-m2r:`<br>`\ shift


How to use it ?
---------------

Here is a simple example showing how to build a 1-Lipschitz network:

.. code-block:: python

    from deel.lip.initializers import BjorckInitializer
    from deel.lip.layers import SpectralDense, SpectralConv2D
    from deel.lip.model import Sequential
    from deel.lip.activations import PReLUlip
    from tensorflow.keras.layers import Input, Lambda, Flatten, MaxPool2D
    from tensorflow.keras import backend as K
    from tensorflow.keras.optimizers import Adam

    # Sequential (resp Model) from deel.model has the same properties as any lipschitz
    # layer ( condense, setting of the lipschitz factor etc...). It act only as a container.
    model = Sequential(
        [
            Input(shape=(28, 28)),
            Lambda(lambda x: K.reshape(x, (-1, 28, 28, 1))),

            # Lipschitz layer preserve the API of their superclass ( here Conv2D )
            # an optional param is available: k_coef_lip which control the lipschitz
            # constant of the layer
            SpectralConv2D(
                filters=32, kernel_size=(3, 3), padding='same',
                activation=PReLUlip(), data_format='channels_last',
                kernel_initializer=BjorckInitializer(15, 50)),
            SpectralConv2D(
                filters=32, kernel_size=(3, 3), padding='same',
                activation=PReLUlip(), data_format='channels_last',
                kernel_initializer=BjorckInitializer(15, 50)),
            MaxPool2D(pool_size=(2, 2), data_format='channels_last'),

            SpectralConv2D(
                filters=64, kernel_size=(3, 3), padding='same',
                activation=PReLUlip(), data_format='channels_last',
                kernel_initializer=BjorckInitializer(15, 50)),
            SpectralConv2D(
                filters=64, kernel_size=(3, 3), padding='same',
                activation=PReLUlip(), data_format='channels_last',
                kernel_initializer=BjorckInitializer(15, 50)),
            MaxPool2D(pool_size=(2, 2), data_format='channels_last'),

            Flatten(),
            SpectralDense(256, activation="relu", kernel_initializer=BjorckInitializer(15, 50)),
            SpectralDense(10, activation="softmax"),
        ],
        k_coef_lip=0.5,
        name='testing'
    )

    optimizer = Adam(lr=0.001)
    model.compile(loss='categorical_crossentropy',
                optimizer=optimizer,
                metrics=['accuracy'])


See :ref:`deel-lip-api` for a complete API description.
