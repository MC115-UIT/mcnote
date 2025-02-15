## Common Patterns


### 1. Factory
	- Nature : Creates objects without explicitly specifying their exacts class.
	- Example : Creating different types of computer components

```c#
    public interface : IComputerComponent
    {
        string GetDescription();
    }

    public class CPU : IComputerComponent
    {
        public string GetDescription() = > "Central processing unit";
    }

    public class GPU : IComputerComponent
    {
        public string GetDescription() = > "Graphics processing unit";
    }

    public class ComputerFactory
    {
        public IComputerComponent CreateComponent(string componentType)
        {
            return componentType.ToLower() switch
            {
                "cpu" => new CPU(),
                "gpu" => new GPU(),
                throw new ArgumentException("Unknown Component type")
            }
        }
    }
    /// usage

    var computerFactory = new ComputerFactory();
    var cpu = computerFactory.CreateComponent("cpu");
    cpu.GetDescription();
```
### 2. Bridge
    - Nature : Seperate an abstraction from its implementation so that the two can vary independently.
    - Example : Different computer types without various operating systems.

```c#
public interface IOperatingSystem
{
	void Boot();
}

public class Windows : IOperatingSystem
{
	public void Boot() => Console.Writeline("Window is working.....");
}

public class Linux : IOperatingSystem
{
	public void Boot() => Console.Writeline("Linux is working...");
}

public abstract class Computer
{
	protected IOperatingSystem _os ;
	protected Computer(IOperatingSystem os)
	{
		_os = os;	
	}
	public abstract void Start();
}

public class Desktop : Computer
{
	public Desktop(IOperatingSystem os) : base(os){}
	public overide void Start()
	{
		Console.WriteLine("Desktop computer is working..");
		_os.Boot();
	}
}
// Usage
var windowsDesktop = new Desktop(new Windows());
windowsDesktop.Start();
```
### 3. Builder
	- Nature : Seperates the construction of a complex object from its representation
	- Example : Building a custom computer with various configurations.

```c#
public class Computer
{
	public string CPU {get;set;}
	public string GPU {get;set;}
	public int RAM {get;set;}
	public int Storage {get;set;}
}

public class ComputerBuilder
{
	private Computer _computer = new Computer();
	public ComputerBuilder WithCPU (string cpu)
	{
		_computer.CPU = cpu;
		return this;
	}
	public ComputerBuilder WithGPU (string gpu)
	{
		_computer.GPU = gpu;
		return this;
	}
	public ComputerBuilder WithRAM (string ram)
	{
		_computer.RAM = ram;
		return this;
	}
	public ComputerBuilder WithStorage (string storage)
	{
		_computer.Storage = storage;
		return this;
	}
	public Computer Build() => _computer;
}
/// Usage
var computer =  new ComputerBuilder()
	.WithCPU("Intel i7")
	.WithGPU("GTX 3080")
	.WithRAM(32)
	.WithStorage(1000)
	.Build();
```
### 4. Composite
	- Nature : Composes object into tree structures to represent part-whole hierarchies.
	- Example : Representing a computer system with its components.

```c#

public abstract class ComputerCompoent
{
	protected string name ;
	protected int price;

	public ComputerComponent(string name, int price)
	{
		this.name = name ;
		this.price = price;
	}

	public abstract int GetPrice();
	public abstract void Display(int depth);
}

public class SingleComponent : ComputerComponent
{
	public SingleComponent(string name , int price) : base(name,price){}
	public override int GetPrice() => price;
	public override void Display(int depth)
	{
		Console.WriteLine(new string("-",depth) + name + ": $" + price);
	}
}

public class CompositeComponent : ComputerComponent
{
	private List<ComputerComponent> _children = new List<ComputerComponent>();
	public CompositeComponent(string name) : base(name,0){}

	public void Add(ComputerComponent computerComponent)
	{
		_children.Add(computerComponent);
	}
	public override int GetPrice()
	{
		return _children.Sum(child => child.GetPrice());
	}
	public override void Display(int depth)
	{
		Console.WriteLine(new string('-', depth) + name + ": $" + GetPrice());
		foreach( var child in _children)
		{
			child.Display(depth + 2);
		}
	}
}

/// Usage

var computer = new CompositeComponent("Computer");
var motherboard = new CompositeComponent("MotherBoard");
motherboard.Add(new SingleComponent("CPU", 300));
motherboard.Add(new SingleComponent("RAM", 200));
computer.Add(motherboard);
computer.Add(new SingleComponent("GPU", 500));

computer.Display(0); 
Console.WriteLine("Total Price: $" + computer.GetPrice());
```

### 5. Chain of responsibility

EXAMPLE : Approving leave request

```c#

	public class LeaveRequest
	{
		public string EmployeeName {get; set;}
		public int NumberOfDays {get; set;}
		public string Reason {get; set;}
	}


	public abstract class LeaveHandler
	{
		protected LeaveHandler NextHandler;

		public void SetNextHandler(LeaveHandler nextHandler)
		{
			NextHandler = nexthandler;
		}

		public void HandleRequest(LeaveRequest leaveRequest)
		{
			if(CanHandle(leaveRequest.NumberOfDays))
			{
				ProcessRequest(leaveRequest);
			}
			else if(NextHanlder != null)
			{
				NextHandler.HandleRequest(leaveRequest);
			}
			else
			{
				Console.WriteLine("Leave request could not be processed.");
			}
		}

		protected abstract bool CanHandle(int numberOfDays);
		protected abstract void ProcessRequest(LeaveRequest request);

	}

	public class TeamLead : LeaveHandler
	{
		protected override bool CanHandle(int numberOfDays) => numberOfDays <= 3;

		protected override void ProcessRequest(LeaveRequest request)
		{
			Console.WriteLine($"Leave request by {request.EmployeeName} for {request.NumberOfDays} days approved by Team Lead.");
		}
	}

	public class ProjectManager : LeaveHandler
	{
		protected override bool CanHandle(int numberOfDays) => numberOfDays <= 5;

		protected override void ProcessRequest(LeaveRequest request)
		{
			Console.WriteLine($"Leave request by {request.EmployeeName} for {request.NumberOfDays} days approved by Project Manager.");
		}
	}

	public class HRManager : LeaveHandler
	{
		protected override bool CanHandle(int numberOfDays) => numberOfDays > 5;

		protected override void ProcessRequest(LeaveRequest request)
		{
			Console.WriteLine($"Leave request by {request.EmployeeName} for {request.NumberOfDays} days approved by HR Manager.");
		}
	}

	// Usage
	using Chainofresponbility;

	// Create handlers
	var teamLead = new TeamLead();
	var projectManager = new ProjectManager();
	var hrManager = new HRManager();

	// Set up the chain
	teamLead.SetNextHandler(projectManager);
	projectManager.SetNextHandler(hrManager);

	// Create a leave request
	var leaveRequest = new LeaveRequest
	{
		EmployeeName = "Devesh",
		NumberOfDays = 4,
		Reason = "Sick Leave"
	};

	// Process the request
	teamLead.HandleRequest(leaveRequest);

	// Another leave request
	var anotherLeaveRequest = new LeaveRequest
	{
		EmployeeName = "Sanjay Singh",
		NumberOfDays = 6,
		Reason = "Vacation"
	};

	// Process the request
	teamLead.HandleRequest(anotherLeaveRequest);
```