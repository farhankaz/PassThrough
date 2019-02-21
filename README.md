# PassThrough
Passthrough Flow for Akka Streams

Implementation a passthrough flow for Akka Streams.  It's common to have a stage in your Akka Streams flow that
takes am effectful action in a context.  For example, you are processing a Kafka message then doing some data proecssing and then
committing a Kafka offset. Since Akka Stream are built as pipelines that transform the data flowing, there is not a simple way
to pass along a 'context' or for example an offset to be committed. Also fan-out shapes like AlsoTo or Broadcast don't by themselves 
do the trick becase you want to ensure the processing happens serially (i.e. affectfully process the message and only then commit the offset).

```text
        ------A => B ------
        |                 |
--- A ---                 ------zip(A, B) => A------
        |                 |
        -------A-----------
```

A sample implementation of how a pass through flow is shown below.

```scala
  object CustomStages {
    import GraphDSL.Implicits._

    type F[T, A] = Flow[T, A, NotUsed]

    object Implicits {
      implicit class PassThroughOps[T, C, A](s:Source[T, A]) {
        def passThrough(f:F[T, C]) :Source[T, A] = s.via(CustomStages.passThrough(f))
        def passThrough(f1:F[T, C], f2:F[T,C]) :Source[T, A] = s.via(CustomStages.passThrough(f1, f2))
      }
    }

    private def zipWith2[T, A] = ZipWith[T, A, T]((o1, _) => o1)
    private def zipWith3[T, A] = ZipWith[T, A, A, T]((o1, _, _) => o1)
    private def zipWith4[T, A] = ZipWith[T, A, A, A, T]((o1, _, _, _) => o1)

    def passThrough[T, A](flow:Flow[T, A, NotUsed]) = build { implicit b =>
      val zip = b.add(zipWith2[T, A])
      val bcast = b.add(Broadcast[T](2))
      bcast ~> zip.in0
      bcast ~> flow ~> zip.in1
      FlowShape(bcast.in, zip.out)
    }

    def passThrough[T, A](flow1:F[T, A], flow2:F[T, A]) = build { implicit b =>
      val zip = b.add(zipWith3[T, A])
      val bcast = b.add(Broadcast[T](3))
      bcast ~> zip.in0
      bcast ~> flow1 ~> zip.in1
      bcast ~> flow2 ~> zip.in2
      FlowShape(bcast.in, zip.out)
    }

    def passThrough[T, A](flow1:F[T, A], flow2:F[T, A], flow3:F[T, A]) = build { implicit b =>
      val zip = b.add(zipWith4[T, A])
      val bcast = b.add(Broadcast[T](4))
      bcast ~> zip.in0
      bcast ~> flow1 ~> zip.in1
      bcast ~> flow2 ~> zip.in2
      bcast ~> flow3 ~> zip.in3
      FlowShape(bcast.in, zip.out)
    }

    def build[T](logic:GraphDSL.Builder[NotUsed] => FlowShape[T, T]) =
      Flow.fromGraph(GraphDSL.create()(logic))
  }
```
