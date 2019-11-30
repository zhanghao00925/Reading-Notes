# Depth Precision Visualized

https://developer.nvidia.com/content/depth-precision-visualized

- Reversed-Z with a float depth buffer gives a zero error rate in this test. Now, of course you can make it generate some errors if you keep tightening the spacing of the input depth values. Still, reversed-Z with float is ridiculously more accurate than any of the other options.
- Reversed-Z with an integer depth buffer is as good as any of the other integer options.
- Reversed-Z erases the distinctions between precomposed versus separate view/projection matrices, and finite versus infinite far planes. In other words, with reversed-Z you can compose your projection matrix with other matrices, and you can use whichever far plane you like, without affecting precision at all.
