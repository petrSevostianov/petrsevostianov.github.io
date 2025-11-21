## Post #1 https://www.reddit.com/r/colorists/

## Solution to the LED Wall Calibration problem of Virtual Production.

Hello! I'm Petr Sevostianov, CTO and co-founder of Antilatency. I have written an article about an important problem in **virtual production** related to cinema cameras and their non-linear color output.

Besides the well-known transfer functions (e.g. SLog3, LogC4 ..) applied by cinema cameras, there is another transform operation: a **non-linear gamut transform**. This transform distorts colors in gamut space and breaks linearity.

Linearity is crucial for virtual production tasks such as lighting calibration, keying, and LED wall to camera matching.

We have learnt how to **measure**, **characterize** this transform.
So we can not only **cancel** this non-linearity out, but also **re-apply** it back at the end of the pipeline to get the original look.

You can read the full article here:
[https://petrsevostianov.github.io/CinemaCameraBeautifiers/](https://petrsevostianov.github.io/CinemaCameraBeautifiers/)

In this article, I describe our methodology, experiments, and findings in detail, complete with images and interactive 3D visualizations.

I hope you find this research useful and comprehensive. Feel free to reach out if you have any questions or would like to discuss further. Join me in my quest to push virtual production towards transparent and reproducible workflows!


## Post #2 https://www.reddit.com/r/videography/

## Chromatic nonlinearity in cinema cameras and its impact on virtual production (specifically LED Wall calibration).

Hi everyone! I'm Petr Sevostianov, CTO at Antilatency, and developer of the CyberGaffer system for virtual production lighting.

Recently, I wrote an [article](https://petrsevostianov.github.io/CinemaCameraBeautifiers/) about a significant challenge in virtual production related to cinema cameras.
During my work on CyberGaffer, I found that it is impossible to calibrate LED walls to cinema cameras accurately, using standard techniques (removing transfer function + matrix 3x3 transform).
The reason is that cinema cameras apply a **non-linear gamut transform** internally, which distorts colors in a non-linear way.
In the article, I describe how we measured and modeled this transform, allowing us to **cancel** it out to get perfectly linear colors and even **re-apply** it in post-processing if we want to restore the original look.

That was a missing piece of knowledge for me, and I believe it is crucial for anyone working in virtual production with cinema cameras.
You can read the full article here: [https://petrsevostianov.github.io/CinemaCameraBeautifiers/](https://petrsevostianov.github.io/CinemaCameraBeautifiers/)
I hope you find this research useful! Feel free to reach out if you have any questions or would like to discuss further. Join me in my quest to push virtual production towards transparent and reproducible workflows!
Thank you!
