[MassTransit.io](https://masstransit.io/documentation/patterns/saga/state-machine)

# Saga Overview

- Saga Pattern, bir dizi event'ı düzenli bir şekilde yönetmek için kullanılan bir pattern'dir.
- Bir saga, bir koordinatör tarafından yönetilen uzun süreli transaction'dır.
- Saga'lar bir event ile başlatılır, sagalar event'ları organize eder ve genel işlemin state'ini korurlar.
- Saga'lar dağıtılan transaction'ların getirdiği complexitiyi locking olmadan ve tutarlı bir şekilde yönetmek için tasarlanmıştır.
- State'i yönetirler ve bir hata durumunda alınacak aksiyonları takip ederler.

## State Machine Sagas

- MassTransit, saga'lar oluşturmak için güçlü bir syntax olarak *state machine*'leri sunar. MassTransit kullanılıyorsa *state machine*'lerin kullanılması kesinlikle önerilir.

## Consumer Sagas

- MassTransit ilişkilendirilmiş saga event'larını consume etmek için bir veya daha fazla interface'i implement eden *consumer sagas* yapısını da destekler.

## Definitions

- Saga tanımlamaları, consumer'ların davranışlarını belirlemek için kullanılır, otomatik olarak yapılandırılabilirler.
- Tanımlamalar `AddSaga` ile açıkça yapılabilir veya `AddSagas`yöntemlerinde birisi kullanılarak otomatik olarak yapılandırılabilir. Örnek olarak basit bir saga tanımlaması:

	```
	public class OrderStateDefinition :
		SagaDefinition<OrderState>
	{
		public OrderStateDefinition()
		{
			// specify the message limit at the endpoint level, which influences
			// the endpoint prefetch count, if supported
			Endpoint(e => e.ConcurrentMessageLimit = 16);
		}

		protected override void ConfigureSaga(IReceiveEndpointConfigurator endpointConfigurator, ISagaConfigurator<OrderState> sagaConfigurator)
		{
			var partition = endpointConfigurator.CreatePartitioner(16);

			sagaConfigurator.Message<SubmitOrder>(x => x.UsePartitioner(partition, m => m.Message.CorrelationId));
			sagaConfigurator.Message<OrderAccepted>(x => x.UsePartitioner(partition, m => m.Message.CorrelationId));
			sagaConfigurator.Message<OrderCanceled>(x => x.UsePartitioner(partition, m => m.Message.CorrelationId));
		}
	}
	```

# Consumer Sagas

- Consumer saga bir class olarak oluşturulur, içerisinde identifier olarak *CorrelationId* değeri olur. Bu değer saga repository içerisindeki state'i tanımlar.
- State ile birlikte saga tarafından handle edilen event'ları tanımlamak için consumer sınıfına interface'ler eklenebilir.
- State ve behavior'ların bir sınıf içerisinde birleşmesine *consumer saga* denir.

## Interfaces

### InitiatedBy

```
public record SubmitOrder :
    CorrelatedBy<Guid>
{
    public Guid CorrelationId { get; init; }
    public DateTime OrderDate { get; init; }
}

public class OrderSaga :
    ISaga,
    InitiatedBy<SubmitOrder>
{
    public Guid CorrelationId { get; set; }

    public DateTime? SubmitDate { get; set; }
    public DateTime? AcceptDate { get; set; }

    public async Task Consume(ConsumeContext<SubmitOrder> context)
    {
        SubmitDate = context.Message.OrderDate;
    }
}
```

- Saga'nın receive endpointi tarafından bir *SubmitOrder* mesajı consume edildiğinde, *CorrelationId* property'si aynı id'ye sahip bir instance olup olmadığını kontrol etmek için kullanılır.			
  Eğer saga repository içerisinde gelen *CorrelationId*'li bir instance bulunamazsa, repository yeni bir instance oluşturur ve instance içerisindeki *Consume* fonksiyonunu çağırır.
  *Consume* fonksiyonu tamamlandıktan sonra saga repository yeni oluşturulan instance'ı kaydeder. 

### Orchestrates

- Var olan bir saga instance'ı tarafından orchestrate edilen bir event tanımlamak için ek olarak interface ve fonksiyon belirtilmelidir. Örnek olarak:

	```
	public record OrderAccepted :
		CorrelatedBy<Guid>
	{
		public Guid CorrelationId { get; init; }
		public DateTime Timestamp { get; init; }
	}

	public class OrderSaga :
		ISaga,
		InitiatedBy<SubmitOrder>,
		Orchestrates<OrderAccepted>,
	{
		public Guid CorrelationId { get; set; }

		public DateTime? SubmitDate { get; set; }
		public DateTime? AcceptDate { get; set; }

		public async Task Consume(ConsumeContext<SubmitOrder> context) {...}

		public async Task Consume(ConsumeContext<OrderAccepted> context)
		{
			AcceptDate = context.Message.Timestamp;
		}
	}
	```

### InitiatedByOrchestrates

- Yeni bir saga instance'ını başlatabilecek veya mevcut bir saga instance'ını yönlendirebilecek bir event tanımlamak için ek bir interface ve metot belirtilmelidir. Örnek olarak:

	```
	public record OrderInvoiced :
		CorrelatedBy<Guid>
	{
		public Guid CorrelationId { get; init; }
		public DateTime Timestamp { get; init; }
		public decimal Amount { get; init; }
	}

	public class OrderPaymentSaga :
		ISaga,
		InitiatedByOrOrchestrates<OrderInvoiced>
	{
		public Guid CorrelationId { get; set; }

		public DateTime? InvoiceDate { get; set; }
		public decimal? Amount { get; set; }

		public async Task Consume(ConsumeContext<OrderInvoiced> context)
		{
			InvoiceDate = context.Message.Timestamp;
			Amount = context.Message.Amount;
		}
	}
	```

### Observes

- Mevcut bir saga instance'ı tarafından gözlemlenen ancak *CorrelatedBy* interface'ini implement etmeyen bir event tanımlamak için ek bir interface ve metot belirtilir. Örnek olarak:

	```
	public record OrderShipped
	{
		public Guid OrderId { get; init; }
		public DateTime ShipDate { get; init; }
	}

	public class OrderSaga :
		ISaga,
		InitiatedBy<SubmitOrder>,
		Orchestrates<OrderAccepted>,
		Observes<OrderShipped, OrderSaga>
	{
		public Guid CorrelationId { get; set; }

		public DateTime? SubmitDate { get; set; }
		public DateTime? AcceptDate { get; set; }
		public DateTime? ShipDate { get; set; }

		public async Task Consume(ConsumeContext<SubmitOrder> context) {...}
		public async Task Consume(ConsumeContext<OrderAccepted> context) {...}

		public async Task Consume(ConsumeContext<OrderShipped> context)
		{
			ShipDate = context.Message.ShipDate;
		}

		public Expression<Func<OrderSaga, OrderShipped, bool>> CorrelationExpression =>
			(saga,message) => saga.CorrelationId == message.OrderId;
	}
	```

## Configuration

- MassTransit'i yapılandırırken bir saga eklemek istiyorsak `AddSaga` fonksiyonunu kullanabiliriz. Örnek olarak:
	```
	services.AddMassTransit(x =>
	{
		x.AddSaga<OrderSaga>()
			.InMemoryRepository();
	});

	```

# Saga State Machines

## Introduction

- *Automatonymous* .NET için bir state machine library'sidir. MassTransit, *Automatonymous* kütüphanesini içerir ve ek özellikler ekler. 
  MassTransit'in eski sürümlerinde *Automatonymous* paketinin uygulamaya ayrı olarak eklenmesi gerekiyordu fakat MassTransit v8 ile birlikte *Automatonymous* kütüphanesi MassTransit içerisine dahil edildi.

### State Machine

- Bir `state machine` sonu olan state akışlarının state'lerini, event'larını ve davranışlarını tanımlar.
- `MassTransitStateMachine<T>` sınıfından kalıtım alan bir class olarak tanımlanan **state machine** bir kez oluşturulur ve sonrasında event'ler ile tetiklenen davranışları **state machine instance**'larına uygulamak için kullanılır.

	```csharp
	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
	}
	```

### Instance

- Bir instance, **state machine instance** için gerekli verileri içerir. 
- Aynı *CorrelationId*'ye sahip mevcut bir instance bulunamadığı zaman, consume edilen her `initial` event'i için yeni bir instance oluşturulur.
- Instance'ları kalıcı olarak saklamak için *saga repository* kullanılır.
- Instance'lar bir class'tır ve `SagaStateMachineInstance` interface'ini implement etmez zorundadır.

	```
	public class OrderState :
		SagaStateMachineInstance
	{
		public Guid CorrelationId { get; set; }
		public string CurrentState { get; set; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			InstanceState(x => x.CurrentState);
		}
	}
	```

- Bir instance *CurrentState* değerini bulundurmak zorundadır. Current state değeri 3 farklı tipte olabilir:
	- `State`: *State* interface'i olarak tanımlanabilir. Serialize etmesi zor olabilir, genellikle sadece in-memory instance'lar için kullanılır.
	- `string`: State ismini tutararak daha kolay kullanılabilir. Her instance içerisinde bulunacağından biraz fazla yer kaplar.
	- `int`: Daha küçük ve daha hızlı fakat her durumun belirtilmesine ve her durum için int değer atanmasına gerek duyulur.

- *CurrentState* değeri *State* olarak tanımlanmış ise otomaik olarak configure edilir. Eğer *string* ve *int* olarak tanımlanırsa `InstanceState` fonksiyonu kullanılmalı. Örnek olarak *CurrentState* *int* olarak şu şekilde kullanılabilir:
	
	```
	public class OrderState :
		SagaStateMachineInstance
	{
		public Guid CorrelationId { get; set; }
		public int CurrentState { get; set; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			InstanceState(x => x.CurrentState, Submitted, Accepted);
		}
	}
	```
	
	Yukarıdaki *CurrentState* değerleri şu şekilde olacaktır: `0-None, 1-Initial, 2-Final, 3-Submitted, 4-Accepted`

### State

- State kavramı, bir instance'ın *CurrentState* durumunda olmasına neden olan daha önce tüketilmiş olayları temsil eder.
- Bir instance herhangi bir anda sadece bir state'e sahip olabilir.
- Yeni bir instance oluşturulduğunda otomatik olarak default değeri *Initial* state olarak oluşur.
- *Final* state'i tüm state machine'ler için de tanımlanır ve bir instance'ın son adıma/duruma geldiğini belirtir.
- Alt kısımdaki örnekte 2 state tanımı var. State'ler *MassTransitStateMachine* base sınıfı tarafından otomatik olarak initialize edilir.
	
	```
	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public State Submitted { get; private set; }
		public State Accepted { get; private set; }
	}
	```

### Event 

- State değerinin değişmesine neden olabilecek şeylere *Event* denir. 
- Bir Event bir instance'a yeni bir data ekleyebilir veya var olan datayı güncelleyebilir. Ayrıca bir instance'ın *CurrentState* değerini değiştirebilir.
- `Event<T>` generiz türdedir ve `T` geçerli bir mesaj tipi olmalıdır.
- Alt kısımdaki örnekte *SubmitOrder* mesajı bir event olarak tanımlanmış ve içerisinde instance ile ilişkilendirilmesini sağlayacak *OrderId* değeri mevcut.
	

	```
	public interface SubmitOrder
	{
		Guid OrderId { get; }    
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => SubmitOrder, x => x.CorrelateById(context => context.Message.OrderId));
		}

		public Event<SubmitOrder> SubmitOrder { get; private set; }
	}

	```

	Event'ler `CorrelatedBy<Guid>` kullanmadığında *correlation expression* kullanmak zorundadır.

### Behavior

- Behavior, instance bir state durumundayken bir event meydana geldiğine yapılacak şeydir.
- Alt kısımdaki örnekte *Initial* durumundayken *SubmitOrder* eventinin davranışını tanımlamak için `Initially` bloğu kullanılıyor.
- Bir *SubmitOrder* mesajı consume edildiğine ve mesaj içerisindeki *OrderId* ile *CorrelationId* değeri eşleşen bir instance bulunamayınca *Initial* stateinde yeni bir instance oluşturulur.
- `TransitionTo` fonksiyonu instance'ı *Submitted* state'ine geçirir, sonrasında instance saga repository kullanılarak kalıcı hale getirilr.

	```
	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Initially(
				When(SubmitOrder)
					.TransitionTo(Submitted));
		}
	}

	```

- Akış olarak örnekler üzerinden devam edersek sonraki işlem olarak *OrderAccepted* event'i alt kısımdaki gibi handle edilmeli:

	```
	public interface OrderAccepted
	{
		Guid OrderId { get; }    
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => OrderAccepted, x => x.CorrelateById(context => context.Message.OrderId));

			During(Submitted,
				When(OrderAccepted)
					.TransitionTo(Accepted));
		}

		public Event<OrderAccepted> OrderAccepted { get; private set; }
	}

	```

#### Message Order

- Mesaj kuyrukları genellikle mesajların sıralı olmasını garanti etmez. State machine oluştururken sıralı olmayan mesajları da düşünerek geliştirme yapılması önemlidir.
- Alt kısımdaki örnekte, *SubmitOrder* mesajının *OrderAccepted* mesajından sonra gelmesi durumunda *SubmitOrder* mesajı *_error* kuyruğuna gönderilecektir.
- *OrderAccepted* mesajı ilk gelseydi, *Initial* durumunda kabul edilmediği için discard edilecekti (gözardı edilecekti).
	
	```
	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Initially(
				When(SubmitOrder)
					.TransitionTo(Submitted),
				When(OrderAccepted)
					.TransitionTo(Accepted));

			During(Submitted,
				When(OrderAccepted)
					.TransitionTo(Accepted));

			During(Accepted,
				Ignore(SubmitOrder));
		}
	}
	```
- Alt kısımdaki güncellenen örnekte *Accepted* durumunda *SubmitOrder* mesajı geldiğine event ignore'lanır. Yine de event içerisindeki data'yı kullanmak isteyebiliriz. Bu tarz durumlarda event içerisindeki datayı instance'a kopyalayabiliriz.

	```
	public interface SubmitOrder
	{
		Guid OrderId { get; }

		DateTime OrderDate { get; }
	}

	public class OrderState :
		SagaStateMachineInstance
	{
		public Guid CorrelationId { get; set; }
		public string CurrentState { get; set; }

		public DateTime? OrderDate { get; set; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Initially(
				When(SubmitOrder)
					.Then(x => x.Saga.OrderDate = x.Message.OrderDate)
					.TransitionTo(Submitted),
				When(OrderAccepted)
					.TransitionTo(Accepted));

			During(Submitted,
				When(OrderAccepted)
					.TransitionTo(Accepted));

			During(Accepted,
				When(SubmitOrder)
					.Then(x => x.Saga.OrderDate = x.Message.OrderDate));
		}
	}

	```

### Configuration

- Bir saga state machine şu şekilde configure edilebilir:

	```
	services.AddMassTransit(x =>
	{
		x.AddSagaStateMachine<OrderStateMachine, OrderState>()
			.InMemoryRepository();
	});

	```

	Bu örnekde in-memory saga repository kullanılıyor fakat herhangi birisi de kullanılabilirdi. Sonraki kısımlarda saga repository'ler ile ilgili detaylar verilecektir.

## Event

- Yukarıda tanımlandığı üzere event kavramı state machine'ler tarafından consume edilen mesajlardır.
- Event'lar geçerli bir mesaj tipini belirtebilir ve her event configure edilebilir. Birkaç farklı event configuration fonksiyonu mevcuttur.
- Built-in olarak gelen `CorrelatedBy<T>` interface'i mesaj contract'ında `CorrelationId` yi belirtmek için kullanılabilir.

	```
	public interface OrderCanceled :
		CorrelatedBy<Guid>
	{    
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => OrderCanceled); // not required, as it is the default convention
		}
	}

	```

	Bu tarz tanımlamalarda `CorrelatedBy<Guid>` interface'ini implement eden event'lar otomatik olarak configure edilir.

- Correlation için event içerisindeki farklı bir alanı tanımlamak istediğimizde alt kısımdaki gibi bir kullanım yapılabilir:

	```
	public interface SubmitOrder
	{    
		Guid OrderId { get; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		// this is shown here, but can be anywhere in the application as long as it executes
		// before the state machine instance is created. Startup, etc. is a good place for it.
		// It only needs to be called once per process.
		static OrderStateMachine()
		{
			GlobalTopology.Send.UseCorrelationId<SubmitOrder>(x => x.OrderId);
		}

		public OrderStateMachine()
		{
			Event(() => SubmitOrder);
		}

		public Event<SubmitOrder> SubmitOrder { get; private set; }
	}
	```

- Event içerisindeki bir alanı correlation için alternatif olarak şu şekilde de tanımlayabiliriz:

	```
	 public interface SubmitOrder
	{    
		Guid OrderId { get; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => SubmitOrder, x => x.CorrelateById(context => context.Message.OrderId));
		}

		public Event<SubmitOrder> SubmitOrder { get; private set; }
	}
	```

- *OrderId* bir *Guid* olduğu sürece correlation için kullanılabilir. *Initial* durumundayken *SubmitOrder* kabul edildiğinde *OrderId* bir *Guid* olduğu için *OrderId* değeri yeni oluşan instance içerisindeki *CorrelationId* alanına otomatik olarak atanır.
- Event'lar *CorrelationId* ile ilişkilendirilmediğinde query expression kullanılarak da ilişkilendirilebilir. Query'ler daha fazla maliyetlidir ve birden çok instance ile eşleşebilir. State machine'leri geliştirirken bu durumları da değerlendirmeliyiz.
- Yine de mümkünse *CorrelationId* üzerinden ilişkilendirme yapmaya çalışalım. Eğer query'ler gerçekten gerekli ise database sorgularını optimizie edebilmek adına ilgili property'ler için index'ler oluşturmak gerekebilir.
- Correlation için farklı bir tür kullanılacaksa alt kısımdaki örneğe benzer bir configuration gerekli olacaktır:

	```
	public interface ExternalOrderSubmitted
	{    
		string OrderNumber { get; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => ExternalOrderSubmitted, e => e
				.CorrelateBy(i => i.OrderNumber, x => x.Message.OrderNumber)
				.SelectId(x => NewId.NextGuid()));
		}

		public Event<ExternalOrderSubmitted> ExternalOrderSubmitted { get; private set; }
	}

	```
- Query'ler saga repository'e direkt olarak gönderilen iki parametre ile de oluşturulabilir:

	```
	public interface ExternalOrderSubmitted
	{    
		string OrderNumber { get; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => ExternalOrderSubmitted, e => e
				.CorrelateBy((instance,context) => instance.OrderNumber == context.Message.OrderNumber)
				.SelectId(x => NewId.NextGuid()));
		}

		public Event<ExternalOrderSubmitted> ExternalOrderSubmitted { get; private set; }
	}

	```

	Event içerisinde instance ile unique olarak ilişki kurulmasını sağlayan bir *Guid* olmadığında `.SelectId` expression kullanılmak zorunda. 						
	
	Yukarıdaki örnekte instance'ın *CorrelationId* alanına atacanak olan değeri oluşturmak için `NewId` kullanılmış.
- *CorrelationId* ile ilişkilendirme yapmak yerine `.SelectId` ile *CorrelationId* oluşturan event'lar, instance duplication olmaması adına property'lerde unique constraint'ler uygulamalı			. 			
	  
	Eğer 2 event aynı property değerine aynı anda bağlanırsa sadece bir tanesi instance'ı depolayabilir, diğeri fail olur (ve eğer yeniden deneme yapılandırılmışsa—ki bir saga kullanıldığında yapılandırılmalıdır—yeniden deneyecektir).
	Bu sırada event instance'ın *CurrentState* durumuna göre yönlendirilir (dispatch edilir). Unique bir constraint kullanılmazsa instance duplication oluşacaktır.
	
- Ek olarak mesaj başlıkları (message headers) da kullanılabilir. Örnek olarak sürekli yeni identifier oluşturmak yerine eğer gönderilmiş ise *CorrelationId* header'ı kullanılabilir. Örnek olarak:
		
	```
	.SelectId(x => x.CorrelationId ?? NewId.NextGuid());
	```
### Ignore Event

- Bazı durumlarda mesajların *_skipped_* kuyruğuna gönderilmesini engellemek için veya hata oluşmasından kaçınmak için event'ların görmezeden gelinmesi gerekebilir.
- Verilen state'de bir event'ı ignore'lamak için `Ignore` fonksiyonu kullanılır. Örnek olarak:

	```
	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Initially(
				When(SubmitOrder)
					.TransitionTo(Submitted),
				When(OrderAccepted)
					.TransitionTo(Accepted));

			During(Submitted,
				When(OrderAccepted)
					.TransitionTo(Accepted));

			During(Accepted,
				Ignore(SubmitOrder));
		}
	}
	```

### Composite Event

- Composite event, tüketilmesi gereken bir veya daha fazla olay belirtilerek yapılandırılır. Bu olaylar tamamlandıktan sonra composite event tetiklenir.
- Required olaran tanımlanan event'ları takip etmek için kullanılacak instance property'si configuration aşamasında belirtilir.
- Composite event tanımlarken önce gerekli olan event'lar ve bu event'ların behavior'ları yapılandırılmalı, sonrasında composite event yapılandırılmalıdır.

	```
	public class OrderState :
		SagaStateMachineInstance
	{
		public Guid CorrelationId { get; set; }
		public string CurrentState { get; set; }

		public int ReadyEventStatus { get; set; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Initially(
				When(SubmitOrder)
					.TransitionTo(Submitted),
				When(OrderAccepted)
					.TransitionTo(Accepted));

			During(Submitted,
				When(OrderAccepted)
					.TransitionTo(Accepted));

			CompositeEvent(() => OrderReady, x => x.ReadyEventStatus, SubmitOrder, OrderAccepted);

			DuringAny(
				When(OrderReady)
					.Then(context => Console.WriteLine("Order Ready: {0}", context.Saga.CorrelationId)));
		}

		public Event OrderReady { get; private set; }
	}
	```

	*SubmitOrder* ve *OrderAccepted* event'ları consume edildikten sonra *OrderReady* event'ı tetiklenir.

- Olayların tanımlanma sırası, yürütülme sıralarını etkileyebilir. Bu nedenle, bileşik olayları diğer tüm olayları ve davranışlarını tanımladıktan sonra state machine'in sonunda tanımlamak en iyisidir.

### Missing Instance

- Eğer bir event herhangi bir instance ile eşleşmez ise, *missing instance behavior* yapılandırılabilir. Örnek olarak:

	```
	public interface RequestOrderCancellation
	{    
		Guid OrderId { get; }
	}

	public interface OrderNotFound
	{
		Guid OrderId { get; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => OrderCancellationRequested, e =>
			{
				e.CorrelateById(context => context.Message.OrderId);

				e.OnMissingInstance(m =>
				{
					return m.ExecuteAsync(x => x.RespondAsync<OrderNotFound>(new { x.OrderId }));
				});
			});
		}

		public Event<RequestOrderCancellation> OrderCancellationRequested { get; private set; }
	}
	```

	Bu örnekte herhangi bir instance ile eşleşmeyen bir siparişi iptal etme isteği consume edildiğinde sipariş bulunamadı yanıtı döndürülecektir.
	Bir hata durumu oluşturmak yerine daha anlamlı bir dönüş sağlanmış olur.
	Missing instance durumunda kullanılabilecek diğer seçenekler *Discard*, *Fault* ve *Execute* işlemleridir.

### Initial Insert

- Yeni instance oluşturma performansını arttırmak için event'i direkt olarak saga repository'a kaydederek lock durumlarını azaltabilecek şekilde yapılandırabiliriz.
- Saga repository'e kaydetme işlemi `Initially` bloğu içerisinde olmalı. Örnek olarak:

	```
	public interface SubmitOrder
	{    
		Guid OrderId { get; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => SubmitOrder, e => 
			{
				e.CorrelateById(context => context.Message.OrderId));

				e.InsertOnInitial = true;
				e.SetSagaFactory(context => new OrderState
				{
					CorrelationId = context.Message.OrderId
				})
			});

			Initially(
				When(SubmitOrder)
					.TransitionTo(Submitted));
		}

		public Event<SubmitOrder> SubmitOrder { get; private set; }
	}
	```

	*InsertOnInitial* kullanırken kritik olan şey saga repository'nin correlation için kullanılacak olan property'nin duplicate olup olmadığını kontrol edebilmesidir.

- Database duplicate'i engelleyebilmek için unique constraint'ler kullanmalıdır.

	```
	public interface ExternalOrderSubmitted
	{    
		string OrderNumber { get; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => ExternalOrderSubmitted, e => 
			{
				e.CorrelateBy(i => i.OrderNumber, x => x.Message.OrderNumber)
				e.SelectId(x => NewId.NextGuid());

				e.InsertOnInitial = true;
				e.SetSagaFactory(context => new OrderState
				{
					CorrelationId = context.CorrelationId ?? NewId.NextGuid(),
					OrderNumber = context.Message.OrderNumber,
				})
			});

			Initially(
				When(SubmitOrder)
					.TransitionTo(Submitted));
		}

		public Event<ExternalOrderSubmitted> ExternalOrderSubmitted { get; private set; }
	}
	```

### Completed Instance

- Default kullanımda instance'lar saga repository'den silinmezler. Tamamlanan instance'ların silinmesini istiyorsak, instance tamamlandığında kullanılacak fonksiyonu belirtmemiz gerekiyor. Örnek olarak:

	```
	public interface OrderCompleted
	{    
		Guid OrderId { get; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => OrderCompleted, x => x.CorrelateById(context => context.Message.OrderId));

			DuringAny(
				When(OrderCompleted)
					.Finalize());

			SetCompletedWhenFinalized();
		}

		public Event<OrderCompleted> OrderCompleted { get; private set; }
	}
	```

	Instance *OrderCompleted* eventini consume ettiğinde sonlandırılır (instance'ı *Final* state'ine geçirir).
	`SetCompletedWhenFinalized` fonksiyonu, *Final* state'indeki instance'ı tamamlanmış olarak (completed) tanımlar. Saga repository tarafında bu tanımlama instance'ın silinmesi için kullanılır.

- Farklı bir completed expression kullanmak istersek, örnek olarak instance'ın state'inin *Completed* olup olmadığını kontrol etmek için `SetCompleted` fonksiyonu alt kısımdaki gibi kullanılabilir:

	```
	public interface OrderCompleted
	{    
		Guid OrderId { get; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => OrderCompleted, x => x.CorrelateById(context => context.Message.OrderId));

			DuringAny(
				When(OrderCompleted)
					.TransitionTo(Completed));

			SetCompleted(async instance => 
			{
				State<TInstance> currentState = await this.GetState(instance);

				return Completed.Equals(currentState);
			});
		}

		public State Completed { get; private set; }
		public Event<OrderCompleted> OrderCompleted { get; private set; }
	}
	```

## Activities

- State machine activities, bir event'a yanıt olarak yürütülen bir dizi aktivite olarka tanımlanır.

### Publish

- Bir event publish etmek için `Publish` aktivitesi kullanılabilir. Örnek olarak:

	```
	public interface OrderSubmitted
	{
		Guid OrderId { get; }    
	}

	public class OrderSubmittedEvent :
		OrderSubmitted
	{
		public OrderSubmittedEvent(Guid orderId)
		{
			OrderId = orderId;
		}

		public Guid OrderId { get; }    
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Initially(
				When(SubmitOrder)
					.Publish(context => (OrderSubmitted)new OrderSubmittedEvent(context.Saga.CorrelationId))
					.TransitionTo(Submitted));
		}
	}
	```

	Alternatif olarak mesaj initializer'da kullanılabilir. Örnek olarak:

	```
	public interface OrderSubmitted
	{
		Guid OrderId { get; }    
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Initially(
				When(SubmitOrder)
					.PublishAsync(context => context.Init<OrderSubmitted>(new { OrderId = context.Saga.CorrelationId }))
					.TransitionTo(Submitted));
		}
	}

	```

### Send

- Bir mesaj göndermek için `Send` aktivitesi kullanılabilir. Örnek olarak:

	```
	public interface UpdateAccountHistory
	{
		Guid OrderId { get; }    
	}

	public class UpdateAccountHistoryCommand :
		UpdateAccountHistory
	{
		public UpdateAccountHistoryCommand(Guid orderId)
		{
			OrderId = orderId;
		}

		public Guid OrderId { get; }    
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine(OrderStateMachineSettings settings)
		{
			Initially(
				When(SubmitOrder)
					.Send(settings.AccountServiceAddress, context => new UpdateAccountHistoryCommand(context.Saga.CorrelationId))
					.TransitionTo(Submitted));
		}
	}
	```

	Alternatif olarak mesaj initializer'da kullanılabilir. Örnek olarak:

	```
	public interface UpdateAccountHistory
	{
		Guid OrderId { get; }    
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine(OrderStateMachineSettings settings)
		{
			Initially(
				When(SubmitOrder)
					.SendAsync(settings.AccountServiceAddress, context => context.Init<UpdateAccountHistory>(new { OrderId = context.Saga.CorrelationId }))
					.TransitionTo(Submitted));
		}
	}
	```

### Respond

- State machine request mesaj tipini bir event olarak yapılandırarak ve `Respond` fonksiyonunu kullanarak request'lere yanıt verebilir.
- Request event'ını yapılandırırken *missing instance* fonksiyonunu kullanmak daha iyi bir response süreci sağlamak adına önerilen bir yöntemdir. Örnek olarak:

	```
	public interface RequestOrderCancellation
	{    
		Guid OrderId { get; }
	}

	public interface OrderCanceled
	{
		Guid OrderId { get; }
	}

	public interface OrderNotFound
	{
		Guid OrderId { get; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Event(() => OrderCancellationRequested, e =>
			{
				e.CorrelateById(context => context.Message.OrderId);

				e.OnMissingInstance(m =>
				{
					return m.ExecuteAsync(x => x.RespondAsync<OrderNotFound>(new { x.OrderId }));
				});
			});

			DuringAny(
				When(OrderCancellationRequested)
					.RespondAsync(context => context.Init<OrderCanceled>(new { OrderId = context.Saga.CorrelationId }))
					.TransitionTo(Canceled));
		}

		public State Canceled { get; private set; }
		public Event<RequestOrderCancellation> OrderCancellationRequested { get; private set; }
	}
	```

- Bazen state machine'den gelecek olan yanıtın beklenmesi gerekebilir. Bu tarz durumlarda orjinal isteğe yanıt vermek için gereken bilgilerin saklanması gerekebilir. Örnek olarak:

	```
	public record CreateOrder(Guid CorrelationId) : CorrelatedBy<Guid>;

	public record ProcessOrder(Guid OrderId, Guid ProcessingId);

	public record OrderProcessed(Guid OrderId, Guid ProcessingId);

	public record OrderCancelled(Guid OrderId, string Reason);

	public class ProcessOrderConsumer : IConsumer<ProcessOrder>
	{
		public async Task Consume(ConsumeContext<ProcessOrder> context)
		{
			await context.RespondAsync(new OrderProcessed(context.Message.OrderId, context.Message.ProcessingId));
		}
	}

	public class OrderState : SagaStateMachineInstance
	{
		public Guid CorrelationId { get; set; }
		public string CurrentState { get; set; }
		public Guid? ProcessingId { get; set; }
		public Guid? RequestId { get; set; }
		public Uri ResponseAddress { get; set; }
		public Guid OrderId { get; set; }
	}

	public class OrderStateMachine : MassTransitStateMachine<OrderState>
	{
		public State Created { get; set; }
    
		public State Cancelled { get; set; }
    
		public Event<CreateOrder> OrderSubmitted { get; set; }
    
		public Request<OrderState, ProcessOrder, OrderProcessed> ProcessOrder { get; set; }
    
		public OrderStateMachine()
		{
			InstanceState(m => m.CurrentState);
			Event(() => OrderSubmitted);
			Request(() => ProcessOrder, order => order.ProcessingId, config => { config.Timeout = TimeSpan.Zero; });

			Initially(
				When(OrderSubmitted)
					.Then(context =>
					{
						context.Saga.CorrelationId = context.Message.CorrelationId;
						context.Saga.ProcessingId = Guid.NewGuid();

						context.Saga.OrderId = Guid.NewGuid();

						context.Saga.RequestId = context.RequestId;
						context.Saga.ResponseAddress = context.ResponseAddress;
					})
					.Request(ProcessOrder, context => new ProcessOrder(context.Saga.OrderId, context.Saga.ProcessingId!.Value))
					.TransitionTo(ProcessOrder.Pending));
        
			During(ProcessOrder.Pending,
				When(ProcessOrder.Completed)
					.TransitionTo(Created)
					.ThenAsync(async context =>
					{
						var endpoint = await context.GetSendEndpoint(context.Saga.ResponseAddress);
						await endpoint.Send(context.Saga, r => r.RequestId = context.Saga.RequestId);
					}),
				When(ProcessOrder.Faulted)
					.TransitionTo(Cancelled)
					.ThenAsync(async context =>
					{
						var endpoint = await context.GetSendEndpoint(context.Saga.ResponseAddress);
						await endpoint.Send(new OrderCancelled(context.Saga.OrderId, "Faulted"), r => r.RequestId = context.Saga.RequestId);
					}),
				When(ProcessOrder.TimeoutExpired)
					.TransitionTo(Cancelled)
					.ThenAsync(async context =>
					{
						var endpoint = await context.GetSendEndpoint(context.Saga.ResponseAddress);
						await endpoint.Send(new OrderCancelled(context.Saga.OrderId, "Time-out"), r => r.RequestId = context.Saga.RequestId);
					}));
		}
	}
	```


### Schedule

- Schedule işlemlerinin sağlanabilmesi için bus'ın *message scheduler*'ı içerecek şekilde yapılandırılması gerekir.
- Bir state machine, bir mesajı instance'a iletmek üzere planlamak için *message scheduler* kullanan event'ları planlayabilir. İlk olarak schedule tanımlanmalı, örnek olarak:

	```
	public interface OrderCompletionTimeoutExpired
	{
		Guid OrderId { get; }
	}

	public class OrderState :
		SagaStateMachineInstance
	{
		public Guid CorrelationId { get; set; }
		public string CurrentState { get; set; }

		public Guid? OrderCompletionTimeoutTokenId { get; set; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			Schedule(() => OrderCompletionTimeout, instance => instance.OrderCompletionTimeoutTokenId, s =>
			{
				s.Delay = TimeSpan.FromDays(30);

				s.Received = r => r.CorrelateById(context => context.Message.OrderId);
			});
		}

		public Schedule<OrderState, OrderCompletionTimeoutExpired> OrderCompletionTimeout { get; private set; }
	}
	```

	Yapılandırmada schedule aktivitesi tarafından override edilebilen `Delay` (gecikme süresi) ve *Received* event'i için correlation expression belirtilir.
	State machine *Received* event'ını consume edebilir.
	`OrderCompletionTimeoutTokenId` alanı `Guid?` olarak tanımlanmıştır ve schedule edilen mesajı, mesajın *tokenId* alanını kullanarak takip etmemizi sağlar. Bu sayede sonrasında bu mesajı unschedule'da edebiliriz.

- Bir event'ı schedule etmek için `Schedule` aktivitesi kullanılabilir, örnek olarak:

	```
	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			During(Submitted,
				When(OrderAccepted)
					.Schedule(OrderCompletionTimeout, context => context.Init<OrderCompletionTimeoutExpired>(new { OrderId = context.Saga.CorrelationId }))
					.TransitionTo(Accepted));
		}
	}
	```

- Eğer schedule edilen event'a artık ihtiyaç yoksa `Unschedule` aktivitesi kullanılabilir, örnek olarak:

	```
	public interface OrderAccepted
	{
		Guid OrderId { get; }    
		TimeSpan CompletionTime { get; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			DuringAny(
				When(OrderCancellationRequested)
					.RespondAsync(context => context.Init<OrderCanceled>(new { OrderId = context.Saga.CorrelationId }))
					.Unschedule(OrderCompletionTimeout)
					.TransitionTo(Canceled));
		}
	}
	```

### Request

- Bir state machine, request tipini ve response tipini belirten `Request` fonksiyonu ile bir request gönderebilir.
- *ServiceAddress* ve *Timeout* dahil olmak üzere ek request ayarları belirtilebilir.
- *ServiceAddress* belirtilirse, bu adres request'e yanıt verecek servisin endpoint adresi olmalıdır. Belirtilmezse request publish edilecektir.
- Default *Timeout* değeri 30 saniyedir fakat `TimeSpan.Zero` değerine eşit veya daha büyük bir değer tanımlanabilir. 
  
  *TimeSpan.Zero* değerinden büyük bir timeout değeri ile istek gönderildiğinde *TimeoutExpired* mesajı schedule edilir. *TimeSpan.Zero* değeri gönderilir ise timeout mesajı schedule edilmez ve istek hiçbir zaman time out olmaz.
  *TimeSpan.Zero* değerinden büyük bir timeout değeri gönerilecek ise bir *message scheduler* yapılandırılmalıdır.
- Bir request tanımlanırken, isteğin *RequestId*'si için bir instance property'si belirlenmelidir. Bu *RequestId*, yanıtların state machine instance'ı ile ilişkilendirilmesini sağlar.
- İstek beklerken, *RequestId* bu property'de saklanır.
- İstek tamamlandığında, property temizlenir.
- İstek time out olursa veya fault olursa, *RequestId* saklanır, bu sayede bu istekler daha sonra tekrar eşleştirilebilir.

#### Configuration

- Bir request tanımlamak için, *Request* property'si eklenmeli ve `Request` fonksiyonu ile yapılandırılmalı. Örnek olarak:

	```
	public interface ProcessOrder
	{
		Guid OrderId { get; }    
	}

	public interface OrderProcessed
	{
		Guid OrderId { get; }
		Guid ProcessingId { get; }
	}

	public class OrderState :
		SagaStateMachineInstance
	{
		public Guid CorrelationId { get; set; }
		public string CurrentState { get; set; }

		public Guid? ProcessOrderRequestId { get; set; }
		public Guid? ProcessingId { get; set; }
	}

	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine(OrderStateMachineSettings settings)
		{
			Request(
				() => ProcessOrder,
				x => x.ProcessOrderRequestId, // Optional
				r => {
					r.ServiceAddress = settings.ProcessOrderServiceAddress;
					r.Timeout = settings.RequestTimeout;
				});
		}

		public Request<OrderState, ProcessOrder, OrderProcessed> ProcessOrder { get; private set; }
	}
	```
- Request aktiviteleri behavior'a eklenebilir, örnek olarak:

	```
	public class OrderStateMachine :
		MassTransitStateMachine<OrderState>
	{
		public OrderStateMachine()
		{
			During(Submitted,
				When(OrderAccepted)
					.Request(ProcessOrder, x => x.Init<ProcessOrder>(new { OrderId = x.Saga.CorrelationId}))
					.TransitionTo(ProcessOrder.Pending));

			During(ProcessOrder.Pending,
				When(ProcessOrder.Completed)
					.Then(context => context.Saga.ProcessingId = context.Message.ProcessingId)
					.TransitionTo(Processed),
				When(ProcessOrder.Faulted)
					.TransitionTo(ProcessFaulted),
				When(ProcessOrder.TimeoutExpired)
					.TransitionTo(ProcessTimeoutExpired));
		}

		public State Processed { get; private set; }
		public State ProcessFaulted { get; private set; }
		public State ProcessTimeoutExpired { get; private set; }
	}
	```
- *Request* 3 event içerir: *Completed, Faulted, TimeoutExpired*. Bu event'lar herhangi bir state'teyken consume edilebilir.
- *Request* ayrıca *Pending* state'ine sahiptir.

#### Missing Instance

- Eğer saga instance'ı response'dan önce sonlandırılırsa, *fault* veya *timeout* alınır, bu tarz durumlar için handler tanımlamaları yapılabilir. Örnek olarak:

	```
	Request(() => ProcessOrder, x => x.ProcessOrderRequestId, r =>
	{
		r.Completed = m => m.OnMissingInstance(i => i.Discard());
		r.Faulted = m => m.OnMissingInstance(i => i.Discard());
		r.TimeoutExpired = m => m.OnMissingInstance(i => i.Discard());
	});
	```

# Persistence

- Saga'lar, state bilgisini koruyan event-based mesaj consumer'larıdır. Event'lar arasında state'i kaydetmek önemlidir. 
  State kalıcı hale getirilmediğinde, bir saga her event'ı yeni bir olay olarak değerlendirecek ve sonraki olayların orchestration'ını bozacak ve hatalı bir duruma yol açacaktır.

- Saga state'ini depolamak için bir tür *sage persistance* yöntemi kullanmamız gerekir. MassTransit içerisinde birkaç farklı persistence yöntemi mevcuttur.

## Order State

- Örnek olarak *OrderState*'i üzerinden konuları işleyebiliriz:

	```
	public class OrderState :
		SagaStateMachineInstance
	{
		public Guid CorrelationId { get; set; }
		public string CurrentState { get; set; }

		public DateTime? OrderDate { get; set; }
	}
	```

## Container Integration

- `AddMassTransit` container extension'ını kullanırken saga registration için bir repository tanımı yapılması gerekir. Örnek olarak in-memory repository kullanımı:
	
	```
	container.AddMassTransit(cfg =>
	{
		cfg.AddSagaStateMachine<OrderStateMachine, OrderState>()
			.InMemoryRepository();
	});
	```

- Saga repository'si her zaman *Singleton* lifecycle olarak register edilir.
- Eğer container registration kullanılmıyor ise saga repository manuel olarak oluşturulabilir ve receive endpoint'inde belirtilebilir. Örnek olarak:

	```
	var orderStateMachine = new OrderStateMachine();
	var repository = new InMemorySagaRepository<OrderState>();

	var busControl = Bus.Factory.CreateUsingInMemory(x =>
	{
		x.ReceiveEndpoint("order-state", e =>
		{
			e.StateMachineSaga(orderStateMachine, repository);
		});
	});
	```

- Saga repository'lerin 2 türü vardır: *Query Repository* ve *Identity-only Repository*.
- Persistence mekanizmasına bağlı olarak, repository uygulaması *Identity-only* veya *identity + query* şeklinde olabilir.
- *Identity-only* repository'i kullanırken (Azure Service Bus veya Redis gibi) yalnızca kimlik ile (identity ile) ilişkilendirme (correlation) kullanabiliriz.
  Bu, saga'nın aldığı bütün event'ların *saga correlation id*'ye sahip olması gerektiğini ve her event için ilişkilendirmenin yalnızca *CorrelateById* yöntemi ile tanımlanabileceği anlamına gelir.

- *Query* repository de tanım gereği kimlik ilişkilendirmesini (identity correlation) destekler, ancak buna ek olarak alınan event'ların diğer property'lerini ve saga state property'lerini de destekler.
  Bu tür ilişkilendirmeleri *CorrelateBy* fonksiyonu ile yapılabilir. Ayrıca bu yöntemde event'ların datalarını ve saga state'lerinin property'lerini kullanarak mantıksal ifadeler de kurulabilir.

## Identity

- Saga instance'ları *Guid* değer ile *CorrelationId* olarak kimliklendirilir. Event'lar saga instance'ları ile ya unique bir *Guid* ile ilişkilendirilir ya da saga instance'ındaki property'leri her event ile ilişkilendiren bir expression aracılığı ile ilişkilendirilir.
  Eğer *CorrelationId* kullanılıyorsa, her zaman 1-1 eşleşme sağlanır: ya saga zaten mevcuttur ya da yeni bir instance oluşturulur.
  Eğer expression kullanılırsa expression birden fazla saga instance'ı ile eşleşebilir. Bu nedenle dikkatli olunmalıdır, çünkü event eşleşen tüm instance'lara iletilecektir.
- *CorrelationId*'nin tabloda key olarak kullanılması şiddetle tavsiye edilir. Bu, daha iyi eşzamanlılık yönetimi sağlayacak ve saga state'inin tutarlılığını koruyacaktır.

## Publishing and Sending From Sagas

- Saga'lar tamamen message-driven durumdalar. Bu nedenle sadece mesaj consume etmezler, yeni event'lar yayınlayabilirler veya yeni command'lar gönderebilirler.
- Eğer bir saga aynı anda çok fazla mesaj alıyorsa ve endpoint birden fazla mesajı paralel olarak işlemeye ayarlanmışsa bu durum mesajların işlenmesi ile saga kalıcılığı arasında bir conflict oluşturabilir.
  Bu, aynı anda kalıcı hale getirilen birden fazla saga state güncellemesi olabileceği anlamına gelir. Saga repository'sinin türüne bağlı olarak bu durum farklı nedenlerle başarısız olabilir (versiyonlama sorunu, satırın veya tablonun lock'lanması veya eTag uyuşmazlığı vb.).
  Bütün bu durumlar aslında şunu belirtir, eşzamanlılık (concurrency) sorunu vardır ve bu durumun çözümü için bir strateji belirlenmelidir.
- Bu tür durumlarda saga repository'nin exception fırlatması normaldir. Ancak eğer saga bir mesaj yayınlıyorsa, bu mesajlar zaten yayınlanmış olabilir fakat saga state'i güncellenememiş olabilir.
  MassTransit enpoint'te retry policy'i uygulayacaktır ve daha da fazla mesaj gönderilecektir, bu da potansiyel olarak bir karmaşaya (*mess*) yol açabilir.
  Èğer bir retry policy tanımlanmadıysa, saga mesajları yayınlar fakat state güncellenmeyeceği için yeni gelen mesajları yanlış state'te olduğundan kabul etmeyebilir.

  Bu yaygın bir sorundur, tüm persistance işlemlerini tamamlayana kadar publish ve send işlemlerini erteleme stratejisi kullanılabilir. Yayınlanması gereken bütün mesajlar *Outbox* isimli buffer içerisinde toplanır. Örnek olarak şöyle kullanılabilir:

	```
	c.ReceiveEndpoint("queue", e =>
	{
		e.UseInMemoryOutbox();
		// other endpoint configuration here
	}
	``` 


## Relational DB Recommendations

- Yeni bir sistem geliştiriyorsak ve Saga DB Entity'i *CorrelationId*'yi primary key olacak şekilde tanımlayabiliyorsak bu harika olur.
- Mevcut bir DB üzerinde çalışıyorsak saga'nın hızlı çalışmasını sağlamak için (optimistic veya pessimistic farketmez) şu adımlara dikkat etmeliyiz:
  
	- *CorrelationId*, tercihen primary key + clustered olmalı.
	- Eğer bu mümkün değilse, clustered index + unique olarak tanımlanmalı. 
	- Ayrıca veritabanı dostu Guid'ler oluşturmak için *NewId* paketinin kullanılması şiddetle tavsiye edilir.


## Optimistic vs pessimistic concurrency

- MassTransit tarafından desteklenen bir çok saga persistance mekanizması, saga'ları işlerken *ACID* garantisi sağlamak için bir yönteme ihtiyaç duyar.Çünkü aynı saga instance'ı için birdne fazla thread
  birden fazla bus event'ını tüketebilir ve bu durum *race condition*'a yol açarak güncellemelerin birbirlerini ezmesine neden olabilir. İlişkisel veritabanları, transaction türünü *serializable* olarak 
  ayarlayarak veya sayfa/satır kilitleme (page/row locking) kullanarak bu durumu kolayca yönetebilir. Bu durum *pessimistic concurrency* olarak adlandırılır.
 
- *Concurrency*'i yönetmenin başka bir yolu, saga her kalıcı hale getirildiğinde version numarasını veya timestamp attribute'unu güncellemektir. Bu durum *optimistic concurrency* olarak adlandırılır.
  Bu yaklaşım veritabanı işlemlerinin başarılı olacağını garanti etmez (bu tür durumlarda retry policy gereklidir). Ancak, tabloyu veya sayfayı kilitlemeyeceği için başkalarının aynı veritabanı ile çalışmasını engellemez.

- Neredeyse her senaryoda *optimistic concurrency* kullanılması önerilir, çünkü çoğu state machine mantığı oldukça hızlı çalışmalıdır.
- Eğer seçilen persistance mekanizması *optimistic concurrency* destekliyorsa, concurrency exception'larına karşı bir retry policy tanımlayarak veya genel bir retry policy tanımlayarak race condition durumu kolayca yönetilebilir.
