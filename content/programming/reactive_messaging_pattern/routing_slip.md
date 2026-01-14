# Routing Slip パターン

## 概要

大規模なビジネス手続きが**論理的には1つのこと**を行うが、**物理的には一連の処理ステップ**を必要とする場合に使用する。これはSOA（Service Oriented Architecture）で一般的に認識されるサービス合成を実現する。各ステップは個々のアクターによって処理される。

```
┌─────────────────────────────────────────────────────────────────┐
│                     Routing Slip パターン                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐               │
│    │ Step │───>│ Step │───>│ Step │───>│ Step │               │
│    │  1   │    │  2   │    │  3   │    │  4   │               │
│    └──────┘    └──────┘    └──────┘    └──────┘               │
│        │           │           │           │                   │
│        v           v           v           v                   │
│    [Actor]     [Actor]     [Actor]     [Actor]                │
│                                                                 │
│    メッセージが各ステップを順番に通過していく                     │
└─────────────────────────────────────────────────────────────────┘
```

## 例：顧客登録プロセス（Enterprise Integration Patterns [EIP]）

| ステップ | 処理内容 |
|---------|---------|
| 1 | 新規顧客を作成 |
| 2 | 顧客の連絡先情報を記録 |
| 3 | 顧客のサービスプランをリクエスト |
| 4 | 新規顧客の与信チェックを実行 |

---

## データ構造

### Value Objects [IDDD]

すべてはイミュータブルな `RegistrationData` Value Object に合成される。

```
┌─────────────────────────────────────────────────────────────────────┐
│                        RegistrationData                             │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────┐  ┌─────────────────────────────────────┐  │
│  │ CustomerInformation │  │       ContactInformation            │  │
│  ├─────────────────────┤  ├─────────────────────────────────────┤  │
│  │ - name: String      │  │  ┌──────────────┐  ┌───────────┐   │  │
│  │ - federalTaxId:     │  │  │PostalAddress │  │ Telephone │   │  │
│  │     String          │  │  ├──────────────┤  ├───────────┤   │  │
│  └─────────────────────┘  │  │- address1    │  │- number   │   │  │
│                           │  │- address2    │  └───────────┘   │  │
│  ┌─────────────────────┐  │  │- city        │                  │  │
│  │   ServiceOption     │  │  │- state       │                  │  │
│  ├─────────────────────┤  │  │- zipCode     │                  │  │
│  │ - id: String        │  │  └──────────────┘                  │  │
│  │ - description:      │  └─────────────────────────────────────┘  │
│  │     String          │                                           │
│  └─────────────────────┘                                           │
└─────────────────────────────────────────────────────────────────────┘
```

```scala
case class CustomerInformation(
    val name: String,
    val federalTaxId: String)

case class ContactInformation(
    val postalAddress: PostalAddress,
    val telephone: Telephone)

case class PostalAddress(
    val address1: String,
    val address2: String,
    val city: String,
    val state: String,
    val zipCode: String)

case class Telephone(val number: String)

case class ServiceOption(
    val id: String,
    val description: String)

case class RegistrationData(
    val customerInformation: CustomerInformation,
    val contactInformation: ContactInformation,
    val serviceOption: ServiceOption)
```

---

## Routing Slip の実装

### ProcessStep と RegistrationProcess

```
┌─────────────────────────────────────────────────────────────────────┐
│                      RegistrationProcess                            │
├─────────────────────────────────────────────────────────────────────┤
│  processId: String                                                  │
│  currentStep: Int (現在のステップ位置)                               │
│                                                                     │
│  processSteps: Seq[ProcessStep]                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ [0]ProcessStep    [1]ProcessStep    [2]ProcessStep    ...   │   │
│  │  ├─ name          ├─ name           ├─ name                 │   │
│  │  └─ processor     └─ processor      └─ processor            │   │
│  │     (ActorRef)       (ActorRef)        (ActorRef)           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  メソッド:                                                          │
│  - isCompleted: Boolean  (currentStep >= processSteps.size)        │
│  - nextStep(): ProcessStep (次のステップを取得)                      │
│  - stepCompleted(): RegistrationProcess (ステップ完了をマーク)       │
└─────────────────────────────────────────────────────────────────────┘
```

各 `ProcessStep` は**名前**と**ステップを実行するアクターへの参照**を持つ。  
`RegistrationProcess` は以下を知っている：
- 各ステップの完了をマークする方法
- プロセス全体が完了したかどうかを判断する方法
- 次に処理されるステップを取得する方法

**設計上、プロセス内のすべてのアクターが同じメッセージ `RegisterCustomer` を受け取る。**

```scala
case class ProcessStep(
    val name: String,
    val processor: ActorRef)

case class RegistrationProcess(
    val processId: String,
    val processSteps: Seq[ProcessStep],
    val currentStep: Int) {

  def this(
      processId: String,
      processSteps: Seq[ProcessStep]) {
    this(processId, processSteps, 0)
  }

  def isCompleted: Boolean = {
    currentStep >= processSteps.size
  }

  def nextStep(): ProcessStep = {
    if (isCompleted) {
      throw new IllegalStateException(
              "Process had already completed.")
    }
    processSteps(currentStep)
  }

  def stepCompleted(): RegistrationProcess = {
    new RegistrationProcess(
            processId,
            processSteps,
            currentStep + 1)
  }
}
```

---

## RegisterCustomer メッセージ

`RegisterCustomer` メッセージは一種の **Envelope Wrapper (314)**。

```
┌─────────────────────────────────────────────────────────────────────┐
│                       RegisterCustomer                              │
│                      (Envelope Wrapper)                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────────┐    ┌──────────────────────┐              │
│   │  registrationData   │    │ registrationProcess  │              │
│   │  (RegistrationData) │    │(RegistrationProcess) │              │
│   └─────────────────────┘    └──────────────────────┘              │
│                                                                     │
│   advance() メソッド:                                               │
│   1. registrationProcess.stepCompleted で currentStep + 1          │
│   2. 完了していなければ、次のステップのアクターに                     │
│      新しい RegisterCustomer を送信                                 │
│   3. RoutingSlip.completedStep() を呼び出し                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

アクターが `RegisterCustomer` メッセージを受信すると：
1. プロセス内の特定のステップを実行するために必要な `RegistrationData` の部分を使用
2. 最後から2番目の操作として、`RegistrationProcess` を次のステップに進め、そのアクターにディスパッチ
3. 各アクターは受信した `RegisterCustomer` メッセージの `advance()` メソッドを使用してこれを行う

`advance()` メソッドは新しいイミュータブルな `RegisterCustomer` メッセージを作成するが、`currentStep` インデックスは実行されたステップの次を指すようにインクリメントされる。

```scala
case class RegisterCustomer(
    val registrationData: RegistrationData,
    val registrationProcess: RegistrationProcess) {

  def advance():Unit = {
    val advancedProcess =
            registrationProcess.stepCompleted
    if (!advancedProcess.isCompleted) {
      advancedProcess.nextStep().processor !
        RegisterCustomer(
                registrationData,
                advancedProcess)
    }
    RoutingSlip.completedStep()
  }
}
```

---

## メッセージフロー全体図

```
┌────────────────────────────────────────────────────────────────────────┐
│                    RegisterCustomer メッセージのフロー                  │
└────────────────────────────────────────────────────────────────────────┘

    [RoutingSlip Driver]
           │
           │ RegisterCustomer(data, process[currentStep=0])
           v
    ┌────────────────┐
    │ CustomerVault  │  Step 1: 顧客作成
    │                │  - customerInformation を使用
    └───────┬────────┘
            │ advance() → 新しい RegisterCustomer(data, process[currentStep=1])
            v
    ┌────────────────┐
    │ ContactKeeper  │  Step 2: 連絡先情報記録
    │                │  - contactInformation を使用
    └───────┬────────┘
            │ advance() → 新しい RegisterCustomer(data, process[currentStep=2])
            v
    ┌────────────────┐
    │ ServicePlanner │  Step 3: サービスプラン選択
    │                │  - serviceOption を使用
    └───────┬────────┘
            │ advance() → 新しい RegisterCustomer(data, process[currentStep=3])
            v
    ┌────────────────┐
    │ CreditChecker  │  Step 4: 与信チェック
    │                │  - federalTaxId を使用
    └───────┬────────┘
            │ advance() → isCompleted = true (これ以上送信しない)
            v
    [Process Completed]
```

---

## ドライバー（RoutingSlip オブジェクト）

`RoutingSlip` オブジェクトは `RegistrationProcess` を `ProcessStep` インスタンスで構成し、`currentStep` を 0（最初のステップ）に初期化する。

```scala
object RoutingSlip extends CompletableApp(4) {
  val processId = java.util.UUID.randomUUID().toString

  val step1 = ProcessStep(
          "create_customer",
          ServiceRegistry.customerVault(
                  system,
                  processId))

  val step2 = ProcessStep(
          "set_up_contact_info",
          ServiceRegistry.contactKeeper(
                  system,
                  processId))

  val step3 = ProcessStep(
          "select_service_plan",
          ServiceRegistry.servicePlanner(
                  system,
                  processId))

  val step4 = ProcessStep(
          "check_credit",
          ServiceRegistry.creditChecker(
                  system,
                  processId))

  val registrationProcess =
      new RegistrationProcess(
              processId,
              Vector(
                step1,
                step2,
                step3,
                step4))

  val registrationData =
      new RegistrationData(
        CustomerInformation(
                "ABC, Inc.", "123-45-6789"),
        ContactInformation(
          PostalAddress(
                  "123 Main Street", "Suite 100",
                  "Boulder", "CO", "80301"),
          Telephone("303-555-1212")),
        ServiceOption(
                "99-1203",
                "A description of 99-1203."))

  val registerCustomer =
      RegisterCustomer(
              registrationData,
              registrationProcess)

  registrationProcess
    .nextStep
    .processor ! registerCustomer

  awaitCompletion
  println("RoutingSlip: is completed.")
}
```

---

## ServiceRegistry

処理ステップアクターを検索するために `RoutingSlip` は `ServiceRegistry` オブジェクトを使用する。

`ServiceRegistry` は特定のアクターがリクエストされるたびに**新しいアクターインスタンスを作成**する。同じアクターインスタンスを再利用することも可能（各1つだけ作成）だが、新規顧客がまれにしか登録されない場合は不要。

**個々のアクターはクリーンアップする必要があり、これは各アクターが簡単に引き受けることができる責任。**

```scala
object ServiceRegistry {
  def contactKeeper(
      system: ActorSystem,
      id: String) = {
    system.actorOf(
            Props[ContactKeeper],
            "contactKeeper-" + id)
  }

  def creditChecker(
          system: ActorSystem,
          id: String) = {
    system.actorOf(Props[CreditChecker],
            "creditChecker-" + id)
  }

  def customerVault(
      system: ActorSystem,
      id: String) = {
    system.actorOf(Props[CustomerVault],
            "customerVault-" + id)
  }

  def servicePlanner(
      system: ActorSystem,
      id: String) = {
    system.actorOf(Props[ServicePlanner],
            "servicePlanner-" + id)
  }
}
```

---

## 個々のアクター

アクターが特定のビジネスロジックの処理を終了するとすぐに、`RegisterCustomer` を次のステップに進めてから**自身を終了**するようにリクエストする。

```scala
class CreditChecker extends Actor {
  def receive = {
    case registerCustomer: RegisterCustomer =>
      val federalTaxId =
              registerCustomer.registrationData
                .customerInformation.federalTaxId

      println(s"CreditChecker: handling register" +
        s"customer to perform credit check: $federalTaxId")

      registerCustomer.advance()

      context.stop(self)
    case message: Any =>
      println(s"CreditChecker: unexpected: $message")
  }
}

class ContactKeeper extends Actor {
  def receive = {
    case registerCustomer: RegisterCustomer =>
      val contactInfo =
              registerCustomer.registrationData
                .contactInformation

      println(s"ContactKeeper: handling register" +
        s"customer to keep contact information: $contactInfo")

      registerCustomer.advance()

      context.stop(self)
    case message: Any =>
      println(s"ContactKeeper: unexpected: $message")
  }
}

class CustomerVault extends Actor {
  def receive = {
    case registerCustomer: RegisterCustomer =>
      val customerInformation =
              registerCustomer.registrationData
                .customerInformation

      println(s"CustomerVault: handling register" +
        s"customer to create a new customer: $customerInformation")

      registerCustomer.advance()

      context.stop(self)
    case message: Any =>
      println(s"CustomerVault: unexpected: $message")
  }
}

class ServicePlanner extends Actor {
  def receive = {
    case registerCustomer: RegisterCustomer =>
      val serviceOption =
              registerCustomer
                .registrationData.serviceOption

      println(s"ServicePlanner: handling register" +
        s"customer to plan a new customer service: $serviceOption")

      registerCustomer.advance()

      context.stop(self)
    case message: Any =>
      println(s"ServicePlanner: unexpected: $message")
  }
}
```

---

## プロセス出力

```
CustomerVault: handling register customer to create
 a new customer:
   CustomerInformation(ABC, Inc.,123-45-6789)
ContactKeeper: handling register customer to keep
 contact information:
   ContactInformation(
     PostalAddress(123 Main Street,Suite 100,Boulder,
     CO,80301),
     Telephone(303-555-1212))
ServicePlanner: handling register customer to plan a
 new customer service:
   ServiceOption(99-1203,A description of 99-1203.)
CreditChecker: handling register customer to perform
 credit check: 123-45-6789
RoutingSlip: is completed.
```

---

## 柔軟性

`RoutingSlip` ドライバーオブジェクトが `RegistrationProcess` を組み立てる方法は変更可能：

- 全体的なプロセスが成功するような**任意の論理的な順序**でステップのシーケンスを配置できる
- ステップを**追加**したり、現在のステップの**間に挿入**したりすることも可能
- Routing Slip の実装は、ステップの進行が Scala の `Seq`（この例では `Vector` として実装）から駆動されるため、**正しく機能し続ける**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         柔軟なステップ構成                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  元の構成:     [Step1] → [Step2] → [Step3] → [Step4]               │
│                                                                     │
│  順序変更可能: [Step2] → [Step1] → [Step4] → [Step3]               │
│                                                                     │
│  挿入可能:     [Step1] → [NewStep] → [Step2] → [Step3] → [Step4]   │
│                                                                     │
│  ※ Seq (Vector) ベースなので柔軟に対応可能                          │
└─────────────────────────────────────────────────────────────────────┘
```
