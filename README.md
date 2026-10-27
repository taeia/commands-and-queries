# Commands and Queries

Companion repository for Chapter 8 of [*Domain-Centric Architecture*](https://github.com/taeia/domain-centric-architecture): the library lending application with its driving ports split into commands and queries, and a second driver that changes the library behind the window's back.

The rules, the aggregates, the repositories, and the domain services are those of Chapter 7, `library.domain` is Chapter 7's to the character, and so are the seventeen scenarios. What changed is the edge of the core. The three driving ports, `ForLendingCopies`, `ForReservingBooks`, and `ForPayingFines`, now carry commands only: each takes the names Chapter 0 uses, a member's name, a copy's number, a book's title, and answers with the few facts the caller's sentence needs, a due date, a position, an amount, never a `Member`, a `Copy`, or a `Loan`. The questions moved to a fourth port, `ForBrowsingTheLibrary`, which answers with records, `CopySummary`, `MemberSummary`, `BookSummary`, and `ReservationSummary`, shaped for a listing and holding dates and amounts as values. A refusal crosses as a `Refusal`, the port's own exception, so that a driver never imports the model's. And the core gained one driven port, `ChangeSignal`, with one method and no arguments, which the application services ring after every completed command.

The window listens on that signal and asks the queries again whenever it rings. To give it something to hear, a new driving adapter, `http`, puts the library on the JDK's built-in HTTP server, and the window's own configurator starts it in the window's program. A `curl` from a terminal borrows a copy, the signal rings, and the tables in the window move.

Each adapter is a Maven module, and each module's `pom.xml` names what it depends on. `core` depends on nothing. `inmemory` and `http` depend on `core`. `desktop`, `cli`, and `scenarios` depend on `core` and `inmemory`, because each one's entry point is the configurator that wires the library, and `desktop` depends on `http` as well, because its configurator starts the endpoint. No driver holds the model: a search for `library.domain` in `desktop`, `cli`, `http`, and `scenarios` finds the configurators, which build `Lending` and `Reserving`; `ConsoleNotifier`, which is a driven adapter; and `Money` in the scenario tests, which is a value.

The library it implements, with its rules and scenarios, is described in [Chapter 0](https://github.com/taeia/domain-centric-architecture/blob/main/chapters/00-the-library.md). The chapter about this code is [Chapter 8](https://github.com/taeia/domain-centric-architecture/blob/main/chapters/08-commands-and-queries.md).

## Running

You need Java 25 and Maven. JavaFX is a Maven dependency, so nothing else has to be installed.

```
mvn install
mvn -pl desktop javafx:run
```

The first command builds all six modules and runs the tests. The second starts the window from the desktop module. Repeat the first command after changing anything outside `desktop`.

A window opens with the library in its starting state, and the status bar says that it also listens on `http://localhost:8080`. The date picker in the toolbar sets the library's idea of today, so you can walk through the scenarios of Chapter 0 and check the due dates and fines by hand. The *Reset library* button returns to the starting state.

Now open a terminal and borrow a copy from there:

```
curl -X POST "http://localhost:8080/borrow?member=John&copy=C1"
```

The answer is the sentence the window would have shown, and the Copies table in the window shows C1 on loan to John without a click. In PowerShell, write `curl.exe`, because `curl` alone is an alias for something else. The five commands are `POST /borrow?member=John&copy=C1`, `POST /return?copy=C1`, `POST /renew?copy=C1`, `POST /reserve?member=Mary&book=The%20Hobbit`, and `POST /pay?member=Jane`; the three listings are `GET /copies`, `GET /members`, and `GET /reservations`. A refusal answers with status 409 and the reason. There is no `day` over HTTP: the window owns the clock.

To start the window from an IDE, run `library.desktop.Launcher`. Running `MainWindow` directly fails with *JavaFX runtime components are missing*, because JavaFX is on the classpath and the Java launcher checks for the modules whenever the main class extends `Application`. `Launcher` does not extend it, so the check is skipped.

The same library on the command line:

```
mvn -q -pl cli exec:java
```

Type `help` for the commands. They are written the way Chapter 0 writes its scenarios: `day 2 March`, `borrow John C1`, `return C1`, `renew C1`, `reserve Mary The Hobbit`, `pay Jane`, and `copies`, `members`, `reservations` to look at the library. `quit` ends the session. A script runs the long scenario of Chapter 0 from start to finish:

```
mvn -q -pl cli exec:java -Dexec.args=cli/scripts/a-month-at-the-library.txt
```

## Tests

```
mvn test
```

`ScenarioTest` lives in the `scenarios` module and has one test per scenario of Chapter 0, with the same name. It holds the four driving ports, names members, copies, and books the way Chapter 0 does, reads the library back through the browsing port, and moves a `SettableClock` from day to day. It never sees an aggregate.

The `core` module has the unit tests of single rules, unchanged from Chapter 7 except that two of the fakes gained a finder. The `cli` module tests the command line: what it prints for what it is told, including the whole month script. The `http` module tests the endpoint: what it answers for what it is sent, and that the change signal rings once per completed command and not for a refusal.

## Layout

```
pom.xml                        the parent: lists the six modules, holds the versions
core/                          the hexagon: domain, application services, and the ports on its edge
  pom.xml                      depends on nothing (JUnit in test scope)
  src/main/java/library/
    domain/                    unchanged from Chapter 7
      Member.java              aggregate root: name, debt, loans; borrow, returnCopy, renew, pay
      Loan.java                entity inside Member: due date, isOverdue, fine, each for a given day
      Queue.java               aggregate root: the reservations of one book; join, leave, first
      Reservation.java         entity inside Queue
      Copy.java                aggregate root: available, on loan, or on hold; lendTo, holdFor, makeAvailable
      Book.java                aggregate root: ISBN and title
      Money.java               value object: an amount in euros, never negative
      Fine.java                value object: days of delay and what they cost
      Lending.java             domain service: borrow, return, renew, across aggregates; asks the clock
      Reserving.java           domain service: reserve
      LibraryException.java    thrown when the library refuses an operation
    application/
      LoanService.java         implements ForLendingCopies: finds the aggregates by name, calls Lending, rings the signal
      ReservationService.java  implements ForReservingBooks: the same for Reserving
      MemberService.java       implements ForPayingFines: the same for Member.pay
      BrowsingService.java     implements ForBrowsingTheLibrary: reads the aggregates into summaries
    ports/
      driving/                 what the outside calls
        ForLendingCopies.java  commands: borrow, returnCopy, renew
        ForReservingBooks.java command: reserve
        ForPayingFines.java    command: pay
        ForBrowsingTheLibrary.java  queries: copies, members, books, reservations
        Borrowed.java          what the three lending commands answer with
        Returned.java
        Renewed.java
        CopySummary.java       what the queries answer with
        CopyStatus.java
        MemberSummary.java
        BookSummary.java
        ReservationSummary.java
        Refusal.java           thrown through a driving port when the library refuses
      driven/                  what the core calls
        MemberRepository.java  all, named, save
        QueueRepository.java
        CopyRepository.java    all, ofBook, numbered, save
        BookRepository.java    all, titled, save
        Clock.java
        Notifier.java
        ChangeSignal.java      one method, no arguments: the library changed
  src/test/java/library/domain/
    MemberTest.java            unit tests of single rules, one class per domain class
    LoanTest.java
    QueueTest.java
    CopyTest.java
    MoneyTest.java
    FineTest.java
    LendingTest.java
    ReservingTest.java
    FakeMemberRepository.java  the driven ports, implemented for these tests
    FakeCopyRepository.java
    FakeQueueRepository.java
    FakeClock.java
inmemory/                      driven adapters
  pom.xml                      depends on core
  src/main/java/library/inmemory/
    InMemoryMemberRepository.java   the repository ports, implemented with lists
    InMemoryQueueRepository.java
    InMemoryCopyRepository.java
    InMemoryBookRepository.java
    SettableClock.java         the clock: set by the date picker, the day command, and the tests
    InMemoryNotifier.java      the notifier: adds each message to a list the window shows
    StartingState.java         the starting state of Chapter 0
http/                          driving adapter: the HTTP endpoint
  pom.xml                      depends on core (inmemory in test scope)
  src/main/java/library/http/
    HttpApi.java               one request becomes one call on a port; started by the desktop configurator
  src/test/java/library/http/
    HttpApiTest.java           what the endpoint answers for what it is sent
desktop/                       driving adapter: the JavaFX window
  pom.xml                      depends on core, inmemory, and http
  src/main/java/library/desktop/
    MainWindow.java            start() is the configurator for the window and the endpoint; refresh() when the signal rings
    Launcher.java              starts MainWindow from an IDE
    Chooser.java               a combo box that shows a record by the text the dialog chooses
    Loans.java                 the copies on loan, described for the Return and Renew dialogs
    BorrowDialog.java          the five dialogs, each holding the ports it needs
    ReturnDialog.java
    RenewDialog.java
    ReserveDialog.java
    PayDialog.java
cli/                           driving adapter: the command line
  pom.xml                      depends on core and inmemory
  scripts/
    a-month-at-the-library.txt the long scenario of Chapter 0 as a script
  src/main/java/library/cli/
    Cli.java                   main() is the configurator; execute() turns a line into a call on a port
    ConsoleNotifier.java       a driven adapter owned by this driver: prints each message
  src/test/java/library/cli/
    CliTest.java               what the command line prints for what it is told
scenarios/                     driving adapter: the tests
  pom.xml                      depends on core and inmemory
  src/test/java/library/scenarios/
    ScenarioTest.java          one test per scenario
```
