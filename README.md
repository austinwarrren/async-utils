# Async Utilities

## Use `ReaderT` for safely converting `Future`-based traits to cats-effect

You have a higher-kinded trait like this:

```scala
trait FooService[F[_]] {
  def foo(i: Int): F[Unit]
}
```

and an implementation in `Future`:

```scala
class FutureFoo extends FooService[Future] {
  def foo(i: Int): Future[Unit] = Future(println(i))
}
```

(Perhaps the implementation was automatically generated by a tool like [Twitter Scrooge](https://github.com/twitter/scrooge).)

With cats-tagless, you can auto-derive a `FunctorK[FooService]` instance and an implementation in `ReaderT[Future, FooService[Future], *]`:

```scala
object FooService {
  import cats.tagless._

  def FooServiceReaderT[F[_]]: FooService[ReaderT[F, FooService[F], *]] = 
    Derive.readerT[FooService, F]
  implicit val FooServiceFunctorK: FunctorK[FooService] = Derive.functorK
}
```

Now you can safely convert your `Future`-based implementation into one in `F[_] : Async`:

```scala
import cats.tagless.syntax.all._
import com.dwolla.util.async._

val futureFoo: FooService[Future] = new FutureFoo

val fooService: FooService[IO] = FooService.FooServiceReaderT[Future].mapK(ScalaFutureService.provide[IO](futureFoo))
```
