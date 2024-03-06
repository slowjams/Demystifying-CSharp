```C#
public interface IObservable<out T>
{
   IDisposable Subscribe(IObserver<T> observer);
}

namespace ConsoleAppDiagnostic
{
    public class Program
    {
        public static async void Main(string[] args)
        {
            var kettle = new Kettle();//初始化热水壶
            var subscribeRef = kettle.Subscribe(new Alter());//订阅

            var boilTask = kettle.StartBoilWaterAsync();//启动开始烧水任务
            var timoutTask = Task.Delay(TimeSpan.FromSeconds(15));//定义15s超时任务
                                                                  //等待，如果超时任务先返回则取消订阅
            var firstReturnTask = await Task.WhenAny(boilTask, timoutTask);
            if (firstReturnTask == timoutTask)
                subscribeRef.Dispose();

            Console.WriteLine("Hello subscriber!");

            Console.ReadLine();
        }
    }

    public class Alter : IObserver<Temperature>
    {
        public void OnCompleted()
        {
            Console.WriteLine("du du du !!!");
        }
        public void OnError(Exception error)
        {
            //Nothing to do
        }
        public void OnNext(Temperature value)
        {
            Console.WriteLine($"{value.Date.ToString()}: Current temperature is {value.Degree}.");
        }
    }

    /// <summary>
    /// 热水壶
    /// </summary>
    public class Kettle : IObservable<Temperature>
    {
        private List<IObserver<Temperature>> observers;
        private decimal temperature = 0;

        public Kettle()
        {
            observers = new List<IObserver<Temperature>>();
        }

        public decimal Temperature
        {
            get => temperature;
            private set
            {
                temperature = value;
                observers.ForEach(observer => observer.OnNext(new Temperature(temperature, DateTime.Now)));

                if (temperature == 100)
                    observers.ForEach(observer => observer.OnCompleted());
            }
        }
        public IDisposable Subscribe(IObserver<Temperature> observer)
        {
            if (!observers.Contains(observer))
            {
                Console.WriteLine("Subscribed!");
                observers.Add(observer);
            }
            //使用UnSubscriber包装，返回IDisposable对象，用于观察者取消订阅
            return new UnSubscriber<Temperature>(observers, observer);
        }
        /// <summary>
        /// 烧水方法
        /// </summary>
        public async Task StartBoilWaterAsync()
        {
            var random = new Random(DateTime.Now.Millisecond);
            while (Temperature < 100)
            {
                Temperature += 10;
                await Task.Delay(random.Next(5000));
            }
        }
    }

    //定义泛型取消订阅对象，用于取消订阅
    internal class UnSubscriber<T> : IDisposable
    {
        private List<IObserver<T>> _observers;
        private IObserver<T> _observer;
        internal UnSubscriber(List<IObserver<T>> observers, IObserver<T> observer)
        {
            this._observers = observers;
            this._observer = observer;
        }
        public void Dispose()
        {
            if (_observers.Contains(_observer))
            {
                Console.WriteLine("Unsubscribed!");
                _observers.Remove(_observer);
            }
        }
    }

    public class Temperature
    {
        public Temperature(decimal temperature, DateTime date)
        {
            Degree = temperature;
            Date = date;
        }
        public decimal Degree { get; }
        public DateTime Date { get; }
    }
}

```