## Kontrolliertes Beenden und Aufräumen

Der Code in Codeblock 20-20 antwortet auf Anfragen asynchron durch die
Verwendung eines Strang-Vorrats (thread pool), wie von uns beabsichtigt. Wir
erhalten einige Warnungen über die Felder `workers`, `id` und `thread`, die wir
nicht direkt benutzen, was uns daran erinnert, dass wir nichts aufräumen. Wenn
wir die weniger elegante Methode <span class="keystroke">Strg+c</span>
verwenden, um den Hauptstrang (main thread) anzuhalten, werden auch alle
anderen Stränge sofort gestoppt, selbst wenn sie gerade dabei sind, eine
Anfrage zu bedienen.

Als Nächstes werden wir das Merkmal (trait) `Drop` implementieren, um `join`
für jeden der Stränge im Vorrat aufzurufen, damit sie die Anfragen, an denen
sie arbeiten, vor dem Schließen beenden können. Dann werden wir einen Weg
implementieren, um den Strängen mitzuteilen, dass sie keine neuen Anfragen mehr
annehmen und herunterfahren sollen. Um diesen Code in Aktion zu sehen, werden
wir unseren Server so modifizieren, dass er nur zwei Anfragen annimmt, bevor er
seinen Strang-Vorrat kontrolliert herunterfährt.

### Implementieren des Merkmals `Drop` auf `ThreadPool`

Lass uns damit beginnen, `Drop` auf unseren Strang-Vorrat zu implementieren.
Wenn der Vorrat aufgeräumt wird, sollten wir auf das Ende unsere Stränge
warten, um sicherzustellen, dass sie ihre Arbeit beenden. Codeblock 20-22 zeigt
einen ersten Versuch einer `Drop`-Implementierung; dieser Code wird noch nicht
ganz funktionieren.

<span class="filename">Dateiname: src/lib.rs</span>

```rust,does_not_compile
# use std::{
#     sync::{mpsc, Arc, Mutex},
#     thread,
# };
#
# pub struct ThreadPool {
#     workers: Vec<Worker>,
#     sender: mpsc::Sender<Job>,
# }
#
# type Job = Box<dyn FnOnce() + Send + 'static>;
#
# impl ThreadPool {
#     /// Erzeuge einen neuen ThreadPool.
#     ///
#     /// Die Größe ist die Anzahl der Stränge im Vorrat.
#     ///
#     /// # Panics
#     ///
#     /// Die Funktion `new` stürzt ab, wenn die Größe Null ist.
#     pub fn new(size: usize) -> ThreadPool {
#         assert!(size > 0);
#
#         let (sender, receiver) = mpsc::channel();
#
#         let receiver = Arc::new(Mutex::new(receiver));
#
#         let mut workers = Vec::with_capacity(size);
#
#         for id in 0..size {
#             workers.push(Worker::new(id, Arc::clone(&receiver)));
#         }
#
#         ThreadPool { workers, sender }
#     }
#
#     pub fn execute<F>(&self, f: F)
#     where
#         F: FnOnce() + Send + 'static,
#     {
#         let job = Box::new(f);
#
#         self.sender.send(job).unwrap();
#     }
# }
#
impl Drop for ThreadPool {
    fn drop(&mut self) {
        for worker in &mut self.workers {
            println!("Worker {} herunterfahren", worker.id);

            worker.thread.join().unwrap();
        }
    }
}
#
# struct Worker {
#     id: usize,
#     thread: thread::JoinHandle<()>,
# }
#
# impl Worker {
#     fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
#         let thread = thread::spawn(move || loop {
#             let job = receiver.lock().unwrap().recv().unwrap();
#
#             println!("Worker {id} hat einen Auftrag erhalten; führe ihn aus.");
#
#             job();
#         });
#
#         Worker { id, thread }
#     }
# }
```

<span class="caption">Codeblock 20-22: Warten auf das Ende der einzelnen
Stränge, wenn der Strang-Vorrat den Gültigkeitsbereich verlässt</span>

Zuerst iterieren wir über alle `workers` im Strang-Vorrat. Wir verwenden dafür
`&mut`, weil `self` eine veränderbare Referenz ist und wir auch in der Lage
sein müssen, `worker` zu verändern. Für jeden `worker` geben wir eine Nachricht
aus, die besagt, dass dieser bestimmte `worker` heruntergefahren wird, und dann
rufen wir auf dem Strang `join` auf. Wenn der Aufruf von `join` fehlschlägt,
benutzen wir `unwrap`, um das Programm abstürzen zu lassen.

Hier ist der Fehler, den wir erhalten, wenn wir diesen Code kompilieren:

```console
$ cargo check
    Checking hello v0.1.0 (file:///projects/hello)
error[E0507]: cannot move out of `worker.thread` which is behind a mutable reference
  --> src/lib.rs:52:13
   |
52 |             worker.thread.join().unwrap();
   |             ^^^^^^^^^^^^^ ------ `worker.thread` moved due to this method call
   |             |
   |             move occurs because `worker.thread` has type `JoinHandle<()>`, which does not implement the `Copy` trait
   |
note: this function takes ownership of the receiver `self`, which moves `worker.thread`
  --> /rustc/d5a82bbd26e1ad8b7401f6a718a9c57c96905483/library/std/src/thread/mod.rs:1581:17

For more information about this error, try `rustc --explain E0507`.
error: could not compile `hello` due to previous error
```

Der Fehler sagt uns, dass wir `join` nicht aufrufen können, weil wir nur eine
veränderbare Ausleihe von jedem `worker` haben und `join` die Eigentümerschaft
für sein Argument übernimmt. Um dieses Problem zu lösen, müssen wir den Strang
`thread` aus der `Worker`-Instanz herausnehmen, damit `join` den Strang
konsumieren kann. Wir haben dies in Codeblock 17-15 getan: Wenn `Worker`
stattdessen ein `Option<Thread::JoinHandle<()>>` hält, können wir die Methode
`take` auf `Option` aufrufen, um den Wert aus der Variante `Some`
herauszuverschieben und eine Variante `None` an ihrer Stelle zu belassen. Mit
anderen Worten, ein `Worker`, der läuft, wird eine Variante `Some` in `thread`
haben, und wenn wir einen `Worker` aufräumen wollen, ersetzen wir `Some` durch
`None`, sodass der `Worker` keinen Strang zum Laufen hat.

Wir wissen also, dass wir die Definition von `Worker` so aktualisieren wollen:

<span class="filename">Dateiname: src/lib.rs</span>

```rust,does_not_compile
# use std::{
#     sync::{mpsc, Arc, Mutex},
#     thread,
# };
#
# pub struct ThreadPool {
#     workers: Vec<Worker>,
#     sender: mpsc::Sender<Job>,
# }
#
# type Job = Box<dyn FnOnce() + Send + 'static>;
#
# impl ThreadPool {
#     /// Erzeuge einen neuen ThreadPool.
#     ///
#     /// Die Größe ist die Anzahl der Stränge im Vorrat.
#     ///
#     /// # Panics
#     ///
#     /// Die Funktion `new` stürzt ab, wenn die Größe Null ist.
#     pub fn new(size: usize) -> ThreadPool {
#         assert!(size > 0);
#
#         let (sender, receiver) = mpsc::channel();
#
#         let receiver = Arc::new(Mutex::new(receiver));
#
#         let mut workers = Vec::with_capacity(size);
#
#         for id in 0..size {
#             workers.push(Worker::new(id, Arc::clone(&receiver)));
#         }
#
#         ThreadPool { workers, sender }
#     }
#
#     pub fn execute<F>(&self, f: F)
#     where
#         F: FnOnce() + Send + 'static,
#     {
#         let job = Box::new(f);
#
#         self.sender.send(job).unwrap();
#     }
# }
#
# impl Drop for ThreadPool {
#     fn drop(&mut self) {
#         for worker in &mut self.workers {
#             println!("Worker {} herunterfahren", worker.id);
#
#             worker.thread.join().unwrap();
#         }
#     }
# }
#
struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}
#
# impl Worker {
#     fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
#         let thread = thread::spawn(move || loop {
#             let job = receiver.lock().unwrap().recv().unwrap();
#
#             println!("Worker {id} hat einen Auftrag erhalten; führe ihn aus.");
#
#             job();
#         });
#
#         Worker { id, thread }
#     }
# }
```

Nun wollen wir uns auf den Compiler stützen, um die anderen Stellen zu finden,
die geändert werden müssen. Wenn wir diesen Code überprüfen, erhalten wir zwei
Fehler:

```console
$ cargo check
    Checking hello v0.1.0 (file:///projects/hello)
error[E0599]: no method named `join` found for enum `Option` in the current scope
  --> src/lib.rs:52:27
   |
52 |             worker.thread.join().unwrap();
   |                           ^^^^ method not found in `Option<JoinHandle<()>>`
   |
note: the method `join` exists on the type `JoinHandle<()>`
  --> /rustc/d5a82bbd26e1ad8b7401f6a718a9c57c96905483/library/std/src/thread/mod.rs:1581:5
help: consider using `Option::expect` to unwrap the `JoinHandle<()>` value, panicking if the value is an `Option::None`
   |
52 |             worker.thread.expect("REASON").join().unwrap();
   |                          +++++++++++++++++

error[E0308]: mismatched types
  --> src/lib.rs:72:22
   |
72 |         Worker { id, thread }
   |                      ^^^^^^ expected enum `Option`, found struct `JoinHandle`
   |
   = note: expected enum `Option<JoinHandle<()>>`
            found struct `JoinHandle<_>`
help: try wrapping the expression in `Some`
   |
72 |         Worker { id, thread: Some(thread) }
   |                      +++++++++++++      +

Some errors have detailed explanations: E0308, E0599.
For more information about an error, try `rustc --explain E0308`.
error: could not compile `hello` due to 2 previous errors
```

Lass uns den zweiten Fehler beheben, der auf den Code am Ende von `Worker::new`
verweist; wir müssen den Wert `thread` in `Some` einpacken, wenn wir einen
neuen `Worker` erstellen. Nimm die folgenden Änderungen vor, um diesen Fehler
zu beheben:

<span class="filename">Dateiname: src/lib.rs</span>

```rust,ignore,does_not_compile
# use std::{
#     sync::{mpsc, Arc, Mutex},
#     thread,
# };
#
# pub struct ThreadPool {
#     workers: Vec<Worker>,
#     sender: mpsc::Sender<Job>,
# }
#
# type Job = Box<dyn FnOnce() + Send + 'static>;
#
# impl ThreadPool {
#     /// Erzeuge einen neuen ThreadPool.
#     ///
#     /// Die Größe ist die Anzahl der Stränge im Vorrat.
#     ///
#     /// # Panics
#     ///
#     /// Die Funktion `new` stürzt ab, wenn die Größe Null ist.
#     pub fn new(size: usize) -> ThreadPool {
#         assert!(size > 0);
#
#         let (sender, receiver) = mpsc::channel();
#
#         let receiver = Arc::new(Mutex::new(receiver));
#
#         let mut workers = Vec::with_capacity(size);
#
#         for id in 0..size {
#             workers.push(Worker::new(id, Arc::clone(&receiver)));
#         }
#
#         ThreadPool { workers, sender }
#     }
#
#     pub fn execute<F>(&self, f: F)
#     where
#         F: FnOnce() + Send + 'static,
#     {
#         let job = Box::new(f);
#
#         self.sender.send(job).unwrap();
#     }
# }
#
# impl Drop for ThreadPool {
#     fn drop(&mut self) {
#         for worker in &mut self.workers {
#             println!("Worker {} herunterfahren", worker.id);
#
#             worker.thread.join().unwrap();
#         }
#     }
# }
#
# struct Worker {
#     id: usize,
#     thread: Option<thread::JoinHandle<()>>,
# }
#
impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        // --abschneiden--

#         let thread = thread::spawn(move || loop {
#             let job = receiver.lock().unwrap().recv().unwrap();
#
#             println!("Worker {id} hat einen Auftrag erhalten; führe ihn aus.");
#
#             job();
#         });
#
        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

Der erste Fehler liegt in unserer `Drop`-Implementierung. Wir haben bereits
erwähnt, dass wir beabsichtigten, `take` auf dem `Option`-Wert aufzurufen, um
`thread` aus `worker` heraus zu verschieben. Die folgenden Änderungen werden
dies tun:

<span class="filename">Dateiname: src/lib.rs</span>

```rust,ignore,not_desired_behavior
# use std::{
#     sync::{mpsc, Arc, Mutex},
#     thread,
# };
#
# pub struct ThreadPool {
#     workers: Vec<Worker>,
#     sender: mpsc::Sender<Job>,
# }
#
# type Job = Box<dyn FnOnce() + Send + 'static>;
#
# impl ThreadPool {
#     /// Erzeuge einen neuen ThreadPool.
#     ///
#     /// Die Größe ist die Anzahl der Stränge im Vorrat.
#     ///
#     /// # Panics
#     ///
#     /// Die Funktion `new` stürzt ab, wenn die Größe Null ist.
#     pub fn new(size: usize) -> ThreadPool {
#         assert!(size > 0);
#
#         let (sender, receiver) = mpsc::channel();
#
#         let receiver = Arc::new(Mutex::new(receiver));
#
#         let mut workers = Vec::with_capacity(size);
#
#         for id in 0..size {
#             workers.push(Worker::new(id, Arc::clone(&receiver)));
#         }
#
#         ThreadPool { workers, sender }
#     }
#
#     pub fn execute<F>(&self, f: F)
#     where
#         F: FnOnce() + Send + 'static,
#     {
#         let job = Box::new(f);
#
#         self.sender.send(job).unwrap();
#     }
# }
#
impl Drop for ThreadPool {
    fn drop(&mut self) {
        for worker in &mut self.workers {
            println!("Worker {} herunterfahren", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}
#
# struct Worker {
#     id: usize,
#     thread: Option<thread::JoinHandle<()>>,
# }
#
# impl Worker {
#     fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
#         let thread = thread::spawn(move || loop {
#             let job = receiver.lock().unwrap().recv().unwrap();
#
#             println!("Worker {id} hat einen Auftrag erhalten; führe ihn aus.");
#
#             job();
#         });
#
#         Worker {
#             id,
#             thread: Some(thread),
#         }
#     }
# }
```

Wie in Kapitel 17 besprochen, nimmt die Methode `take` auf `Option` die
Variante `Some` heraus und lässt an ihrer Stelle `None` stehen. Wir benutzen
`if let`, um die `Some` zu destrukturieren und den Strang zu erhalten; dann
rufen wir `join` auf dem Strang auf. Wenn der Strang eines `Worker` bereits
`None` ist, wissen wir, dass der Strang bereits aufgeräumt wurde, also passiert
in diesem Fall nichts.

### Den Strängen signalisieren, nicht mehr nach Aufträgen zu lauschen

Mit all den Änderungen, die wir vorgenommen haben, lässt sich unser Code ohne
jede Warnung kompilieren. Aber die schlechte Nachricht ist, dass dieser Code
noch nicht so funktioniert, wie wir es uns wünschen. Der Schlüssel ist die
Logik in den Funktionsabschlüssen, die von den Strängen der `Worker`-Instanzen
ausgeführt werden: Im Moment rufen wir `join` auf, aber das wird die Stränge
nicht herunterfahren, weil sie sich in einer Endlosschleife auf der Suche nach
Aufträgen befinden. Wenn wir versuchen, unseren `ThreadPool` mit unserer
aktuellen Implementierung von `Drop` aufräumen zu lassen, wird der Hauptstrang
für immer blockieren und auf das Beenden des ersten Strangs warten.

Um dieses Problem zu beheben, brauchen wir eine Änderung in der Implementierung
von `drop` in `ThreadPool` und dann eine Änderung in der `Worker`-Schleife.

Zuerst ändern wir die Implementierung von `drop` in `ThreadPool`, um den
`Sender` explizit zu aufzuräumen, bevor wir auf das Ende der Stränge warten.
Codeblock 20-23 zeigt die Änderungen an `ThreadPool`, um den `Absender`
explizit aufzuräumen. Wir verwenden die gleiche `Option` und `take`-Technik wie
beim Strang, um `sender` aus dem `ThreadPool` herauszuverschieben:

<span class="filename">Dateiname: src/lib.rs</span>

```rust
# use std::{
#     sync::{mpsc, Arc, Mutex},
#     thread,
# };
#
pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}
// --abschneiden--
#
# type Job = Box<dyn FnOnce() + Send + 'static>;
#
impl ThreadPool {
#     /// Erzeuge einen neuen ThreadPool.
#     ///
#     /// Die Größe ist die Anzahl der Stränge im Vorrat.
#     ///
#     /// # Panics
#     ///
#     /// Die Funktion `new` stürzt ab, wenn die Größe Null ist.
#     pub fn new(size: usize) -> ThreadPool {
          // --abschneiden--

#         assert!(size > 0);
#
#         let (sender, receiver) = mpsc::channel();
#
#         let receiver = Arc::new(Mutex::new(receiver));
#
#         let mut workers = Vec::with_capacity(size);
#
#         for id in 0..size {
#             workers.push(Worker::new(id, Arc::clone(&receiver)));
#         }
#
        ThreadPool {
            workers,
            sender: Some(sender),
        }
#     }
#
    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);

        self.sender.send(Message::NewJob(job)).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        drop(self.sender.take());

        for worker in &mut self.workers {
            println!("Worker {} herunterfahren", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}
#
# struct Worker {
#     id: usize,
#     thread: Option<thread::JoinHandle<()>>,
# }
#
# impl Worker {
#     fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) -> Worker {
#         let thread = thread::spawn(move || loop {
#             let job = receiver.lock().unwrap().recv().unwrap();
#          
#             println!("Worker {id} got a job; executing.");
#          
#             job();
#         });
#
#         Worker {
#             id,
#             thread: Some(thread),
#         }
#     }
# }
```

<span class="caption">Codeblock 20-23: `sender` vor dem Warten auf die
`Worker`-Stränge explizit aufräumen</span>

Das Aufräumen von `sender` schließt den Kanal, was bedeutet, dass keine
weiteren Nachrichten gesendet werden. Wenn das passiert, geben alle Aufrufe
von `recv`, die die `Worker` in der Endlosschleife machen, einen Fehler zurück.
In Codeblock 20-24 ändern wir die `Worker`-Schleife so, dass die Schleife in
diesem Fall ordnungsgemäß beendet wird, was bedeutet, dass die Stränge beendet
werden, wenn die Implementierung von `drop` in `ThreadPool` `join` für sie
aufruft.

<span class="filename">Dateiname: src/lib.rs</span>

```rust,noplayground
# use std::{
#     sync::{mpsc, Arc, Mutex},
#     thread,
# };
#
# pub struct ThreadPool {
#     workers: Vec<Worker>,
#     sender: mpsc::Sender<Message>,
# }
#
# type Job = Box<dyn FnOnce() + Send + 'static>;
#
# impl ThreadPool {
#     /// Erzeuge einen neuen ThreadPool.
#     ///
#     /// Die Größe ist die Anzahl der Stränge im Vorrat.
#     ///
#     /// # Panics
#     ///
#     /// Die Funktion `new` stürzt ab, wenn die Größe Null ist.
#     pub fn new(size: usize) -> ThreadPool {
#         assert!(size > 0);
#
#         let (sender, receiver) = mpsc::channel();
#
#         let receiver = Arc::new(Mutex::new(receiver));
#
#         let mut workers = Vec::with_capacity(size);
#
#         for id in 0..size {
#             workers.push(Worker::new(id, Arc::clone(&receiver)));
#         }
#
#         ThreadPool {
#             workers,
#             sender: Some(sender),
#         }
#     }
#
#     pub fn execute<F>(&self, f: F)
#     where
#         F: FnOnce() + Send + 'static,
#     {
#         let job = Box::new(f);
#
#         self.sender.send(Message::NewJob(job)).unwrap();
#     }
# }
#
# impl Drop for ThreadPool {
#     fn drop(&mut self) {
#         drop(self.sender.take());
#
#         for worker in &mut self.workers {
#             println!("Worker {} herunterfahren", worker.id);
#
#             if let Some(thread) = worker.thread.take() {
#                 thread.join().unwrap();
#             }
#         }
#     }
# }
#
# struct Worker {
#     id: usize,
#     thread: Option<thread::JoinHandle<()>>,
# }
#
impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            match receiver.lock().unwrap().recv() {
                Ok(job) => {
                    println!("Worker {id} hat einen Auftrag erhalten; führe ihn aus.");

                    job();
                }
                Err(_) => {
                    println!("Worker {id} nicht mehr verbunden, wird beendet.");
                    break;
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

<span class="caption">Codeblock 20-24: Explizites Verlassen der Schleife, wenn
`recv` einen Fehler zurückgibt</span>

Um diesen Code in Aktion zu sehen, modifizieren wir `main` so, dass nur zwei
Anfragen akzeptiert werden, bevor der Server kontrolliert heruntergefahren
wird, wie in Codeblock 20-25 gezeigt.

<span class="filename">Dateiname: src/main.rs</span>

```rust,noplayground
# use hello::ThreadPool;
# use std::fs;
# use std::io::prelude::*;
# use std::net::TcpListener;
# use std::net::TcpStream;
# use std::thread;
# use std::time::Duration;
#
fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming().take(2) {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }

    println!("Fahre herunter.");
}
# 
# fn handle_connection(mut stream: TcpStream) {
#     let mut buffer = [0; 1024];
#     stream.read(&mut buffer).unwrap();
#
#     let get = b"GET / HTTP/1.1\r\n";
#     let sleep = b"GET /sleep HTTP/1.1\r\n";
#
#     let (status_line, filename) = if buffer.starts_with(get) {
#         ("HTTP/1.1 200 OK", "hello.html")
#     } else if buffer.starts_with(sleep) {
#         thread::sleep(Duration::from_secs(5));
#         ("HTTP/1.1 200 OK", "hello.html")
#     } else {
#         ("HTTP/1.1 404 NOT FOUND", "404.html")
#     };
#
#     let contents = fs::read_to_string(filename).unwrap();
#
#     let response = format!(
#         "{}\r\nContent-Length: {}\r\n\r\n{}",
#         status_line,
#         contents.len(),
#         contents
#     );
#
#     stream.write(response.as_bytes()).unwrap();
#     stream.flush().unwrap();
# }
```

<span class="caption">Codeblock 20-25: Herunterfahren des Servers, nachdem er
zwei Anfragen bearbeitet hat, durch Verlassen der Schleife</span>

Du würdest nicht wollen, dass ein Webserver aus der realen Welt
heruntergefahren wird, nachdem er nur zwei Anfragen bearbeitet hat. Dieser Code
zeigt nur, dass das kontrollierte Herunterfahren und Aufräumen funktioniert.

Die Methode `take` ist im Merkmal `Iterator` definiert und beschränkt die
Iteration auf die ersten beiden Elemente. Der `ThreadPool` wird am Ende von
`main` den Gültigkeitsbereich verlassen und die `drop`-Implementierung
ausgeführt werden.

Starte den Server mit `cargo run` und stelle drei Anfragen. Die dritte Anfrage
sollte fehlerhaft sein und in deinem Terminal solltest du eine ähnliche Ausgabe
wie diese sehen:

```console
$ cargo run
   Compiling hello v0.1.0 (file:///projects/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 1.0s
     Running `target/debug/hello`
Worker 0 hat einen Auftrag erhalten; führe ihn aus.
Fahre herunter.
Worker 0 herunterfahren
Worker 3 hat einen Auftrag erhalten; führe ihn aus.
Worker 1 nicht mehr verbunden, wird beendet.
Worker 2 nicht mehr verbunden, wird beendet.
Worker 3 nicht mehr verbunden, wird beendet.
Worker 0 nicht mehr verbunden, wird beendet.
Worker 1 herunterfahren
Worker 2 herunterfahren
Worker 3 herunterfahren
```

Möglicherweise siehst du eine andere Reihenfolge der `Worker` und der
ausgegebenen Nachrichten. Wir können anhand der Nachrichten sehen, wie dieser
Code funktioniert: Die `Worker` 0 und 3 haben die ersten beiden Anfragen
erhalten. Der Server hat nach der zweiten Verbindung aufgehört, Verbindungen
anzunehmen, und die `Drop`-Implementierung auf `ThreadPool` beginnt mit der
Ausführung, bevor `Worker` 3 überhaupt seinen Job beginnt. Wenn man den
`sender` aufräumt, werden alle `Worker` getrennt und angewiesen, sich zu
beenden. Die `Worker` geben jeweils eine Nachricht aus, wenn sie die Verbindung
trennen, und dann ruft der Strang-Vorrat `join` auf, um das Ende jedes
`Worker`-Strangs zu warten.

Beachte einen interessanten Aspekt diesem speziellen Programmlauf: Der
`ThreadPool` hat den `sender` aufgeräumt, und bevor ein `Worker` einen Fehler
erhalten hat, haben wir versucht, auf `Worker` 0 zu warten. `Worker` 0 hatte
noch keinen Fehler von `recv` erhalten, also blockierte der Hauptstrang und
wartete darauf, dass `Worker` 0 fertig wird. In der Zwischenzeit erhielt
`Worker` 3 einen Auftrag, und dann erhielten alle Stränge einen Fehler. Als
`Worker` 0 fertig war, wartete der Hauptstrang darauf, dass die restlichen
`Worker` fertig wurden. Zu diesem Zeitpunkt hatten sie alle ihre Schleifen
verlassen und konnten sich beenden.

Herzlichen Glückwunsch! Wir haben jetzt unser Projekt abgeschlossen; wir haben
einen einfachen Webserver, der einen Strang-Vorrat verwendet, um asynchron zu
antworten. Wir sind in der Lage, den Server kontrolliert herunterzufahren,
wodurch alle Stränge im Vorrat aufgeräumt werden.

Hier ist der vollständige Code als Referenz:

<span class="filename">Dateiname: src/main.rs</span>

```rust,ignore
use hello::ThreadPool;
use std::fs;
use std::io::prelude::*;
use std::net::TcpListener;
use std::net::TcpStream;
use std::thread;
use std::time::Duration;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }

    println!("Fahre herunter.");
}

fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 1024];
    stream.read(&mut buffer).unwrap();

    let get = b"GET / HTTP/1.1\r\n";
    let sleep = b"GET /sleep HTTP/1.1\r\n";

    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK", "hello.html")
    } else if buffer.starts_with(sleep) {
        thread::sleep(Duration::from_secs(5));
        ("HTTP/1.1 200 OK", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };

    let contents = fs::read_to_string(filename).unwrap();

    let response = format!(
        "{}\r\nContent-Length: {}\r\n\r\n{}",
        status_line,
        contents.len(),
        contents
    );

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

<span class="filename">Dateiname: src/lib.rs</span>

```rust
use std::{
    sync::{mpsc, Arc, Mutex},
    thread,
};

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}

type Job = Box<dyn FnOnce() + Send + 'static>;

impl ThreadPool {
    /// Erzeuge einen neuen ThreadPool.
    ///
    /// Die Größe ist die Anzahl der Stränge im Vorrat.
    ///
    /// # Panics
    ///
    /// Die Funktion `new` stürzt ab, wenn die Größe Null ist.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool {
            workers,
            sender: Some(sender),
        }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);

        self.sender.send(Message::NewJob(job)).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        drop(self.sender.take());

        for worker in &mut self.workers {
            println!("Worker {} herunterfahren", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let message = receiver.lock().unwrap().recv().unwrap();

            match message {
                Ok(job) => {
                    println!("Worker {} hat einen Auftrag erhalten; führe ihn aus.", id);

                    job();
                }
                Err(_) => {
                    println!("Worker {} nicht mehr verbunden, wird beendet.", id);
                    break;
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

Wir könnten hier mehr tun! Wenn du dieses Projekt weiter verbessern willst,
findest du hier einige Ideen:

* Füge weitere Dokumentation zu `ThreadPool` und seinen öffentlichen Methoden
  hinzu.
* Füge Tests der Funktionalität der Bibliothek hinzu.
* Ändere Aufrufe von `unwrap` in eine robustere Fehlerbehandlung.
* Verwende `ThreadPool`, um eine andere Aufgabe als das Beantworten von
  Web-Anfragen durchzuführen.
* Suche eine Strang-Vorrats-Kiste auf [crates.io](https://crates.io/) und
  implementiere damit einen ähnlichen Webserver unter Verwendung der Kiste.
  Vergleiche dann dessen API und Robustheit mit dem von uns implementierten
  Strang-Vorrat.

## Zusammenfassung

Gut gemacht! Du hast es bis ans Ende des Buchs geschafft! Wir möchten dir
danken, dass du uns auf dieser Tour durch Rust begleitet hast. Du bist nun
bereit, deine eigenen Rust-Projekte umzusetzen und bei den Projekten anderer zu
helfen. Denke daran, dass es eine gastfreundliche Gemeinschaft von anderen
Rust-Entwicklern gibt, die dir bei allen Herausforderungen, denen du auf deiner
Rust-Reise begegnest, gerne helfen würden.
