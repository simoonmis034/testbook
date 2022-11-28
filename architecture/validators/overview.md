# Overview

Anatomy of a Validator

![](https://docs.solana.com/assets/images/validator-64cad642fda2dc0de9cd1b695651e620.svg)Validator block diagrams

## Pipelining <a href="#pipelining" id="pipelining"></a>

The validators make extensive use of an optimization common in CPU design, called _pipelining_.&#x20;

Pipelining is the right tool for the job when there's a stream of input data that needs to be processed by a sequence of steps, and there's different hardware responsible for each.&#x20;

The quintessential example is using a washer and dryer to wash/dry/fold several loads of laundry. Washing must occur before drying and drying before folding, but each of the three operations is performed by a separate unit.&#x20;

To maximize efficiency, one creates a pipeline of _stages_.&#x20;

We'll call the washer one stage, the dryer another, and the folding process a third. To run the pipeline, one adds a second load of laundry to the washer just after the first load is added to the dryer.&#x20;

Likewise, the third load is added to the washer after the second is in the dryer and the first is being folded. In this way, one can make progress on three loads of laundry simultaneously.&#x20;

Given infinite loads, the pipeline will consistently complete a load at the rate of the slowest stage in the pipeline.

## Pipelining in the Validator <a href="#pipelining-in-the-validator" id="pipelining-in-the-validator"></a>

The validator contains two pipelined processes, one used in leader mode called the TPU and one used in validator mode called the TVU.&#x20;

In both cases, the hardware being pipelined is the same, the network input, the GPU cards, the CPU cores, writes to disk, and the network output.&#x20;

What it does with that hardware is different.&#x20;

The TPU exists to create ledger entries whereas the TVU exists to validate them.
