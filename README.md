Your question is about countdown timers, specifically:
>**How can I put different countdown timers in different cells?**

It seems likely that for any given cell in your `DataGridView` the information that it needs can be calculated by knowing these three things:
- What is the input time?
- What is the output time?
- _What time is is **NOW**!?_

In an open ended way, you have asked for "help with this problem" so I will offer you one possible solution because I believe that using the `DataSource` property of `DataGridView` will help you simplify all of your interactions with this powerful UI control. This will also prove out the assertion that it's "sufficient to have one timer". Its purpose will be to provide continuous updated information for _what time is it now_.  

[![screenshot][1]][1]

***
**Record `class` represents a Row**

Consider making a class that is a "model" for a row. Add public properties for values you wish to be shown in your `DataGridView`. Obtaining a result similar to what you have shown in your post is simplified by giving it the ability to update _itself_ based on the current time.

    enum State{ WAITING, ACTIVE, EXPIRED, FREE, }
    class Record
    {
        [DisplayName("ID code")]
        public string IdCode { get; set; } = string.Empty;

        [DisplayName("Input time")]
        public DateTime? InputTime { get; set; }

        [DisplayName("Output time")]
        public DateTime? OutputTime { get; set; }

        // Record calculates itself when the DataGridView refreshes.
        public string Remaining
        {
            get
            {
                if((InputTime == null) || (OutputTime == null))
                {
                    State = State.FREE;
                    return string.Empty;
                }
                else
                {
                    var now = DateTime.Now;
                    // Are we inside the time window?
                    if((InputTime <= now) && (now <= OutputTime))
                    {
                        State = State.ACTIVE;
                        return (OutputTime - now)?.ToString(@"hh\:mm\:ss")!;
                    }
                    else
                    {
                        if(InputTime > now)
                        {
                            State = State.WAITING;
                            return (OutputTime - InputTime)?.ToString(@"hh\:mm")!;
                        }
                        else
                        {
                            State = State.EXPIRED;
                            return "0";
                        }
                    }
                }
            }
        }
        public State State { get; private set; }
    }

***
**Data Binding**

Using the `Record` class, assign the `dataGridView.DataSource` to a `BindingList<Record>()`. Now, instead of having to work with the UI control directly you can add or remove records by manipulating the Records list.

***
**Updates**

Whenever `Refresh` is called on the `DataGridView` the individual records will refresh their calculations. To ensure that this refresh occurs, use the method that loads the main form to perform the following:

- Auto-generate columns for the DGV.
- Add four items for testing purposes.
- Attach a timer event to update the DGV and the main form title bar.

Once this is done, the view will globally refresh one time per second, giving you a behavior similar to what you have described.

    public partial class MainForm : Form
    {
        public MainForm() => InitializeComponent();
        protected override void OnLoad(EventArgs e)
        {
            base.OnLoad(e);
            dataGridView.DataSource = Records;

            #region F O R M A T    C O L U M N S
            Records.Add(new Record()); // <- Auto-generate columns
            foreach (DataGridViewColumn init in dataGridView.Columns)
            {
                switch (init.Name)
                {
                    case nameof(Record.IdCode):
                        init.AutoSizeMode = DataGridViewAutoSizeColumnMode.Fill;
                        break;
                    default:
                        init.AutoSizeMode = DataGridViewAutoSizeColumnMode.AllCells;
                        break;
                }
            }
            dataGridView.Columns[nameof(Record.IdCode)].AutoSizeMode = DataGridViewAutoSizeColumnMode.Fill;
            dataGridView.Columns[nameof(Record.InputTime)].DefaultCellStyle.Format = "dd/MM/yyyy HH:mm";
            dataGridView.Columns[nameof(Record.OutputTime)].DefaultCellStyle.Format = "dd/MM/yyyy HH:mm";
            dataGridView.CellPainting += onDgvCellPainting;
            Records.Clear();
            #endregion F O R M A T    C O L U M N S

            addTestRecords();

            // Because we're using System.Windows.Forms.Timer the
            // ticks are issued on the UI thread. Invoke not required.
            _seconds.Tick += onSecondsTick;
            _seconds.Start();
        }
Refresh the title bar and the DGV when the seconds timer fires event.

        BindingList<Record> Records { get; } =
            new BindingList<Record>();
        private void onSecondsTick(object sender, EventArgs e)
        {
            Text = $"Main Form - {DateTime.Now.ToLongTimeString()}";
            dataGridView.Refresh();
        }
        System.Windows.Forms.Timer _seconds = 
            new System.Windows.Forms.Timer { Interval = 1000 };


Convert the `State` property to a corresponding cell `BackColor`.

        private void onDgvCellPainting(object sender, DataGridViewCellPaintingEventArgs e)
        {
            if (e.ColumnIndex.Equals(dataGridView.Columns["State"].Index) && (e.RowIndex != -1))
            {
                switch (Records[e.RowIndex].State)
                {
                    case State.WAITING: e.CellStyle.BackColor = Color.LightBlue; break;
                    case State.ACTIVE: e.CellStyle.BackColor = Color.LightYellow; break;
                    case State.EXPIRED: e.CellStyle.BackColor = Color.LightGray; break;
                    case State.FREE: e.CellStyle.BackColor = Color.LightGreen; break;
                }
            }
        }

Add a few records so the behavior can be evaluated.

        // Simulated records for minimal sample.
        private void addTestRecords()
        {
            var now = DateTime.Now;
            // For testing, round to whole minutes;
            now = new DateTime(now.Year, now.Month, now.Day, now.Hour, now.Minute, 0);

            Records.Add(new Record
            {
                IdCode = "fix0001",
                InputTime = now - TimeSpan.FromHours(1),
                OutputTime = now - TimeSpan.FromHours(1) + TimeSpan.FromMinutes(10),
            });
            Records.Add(new Record
            {
                IdCode = "fix0002",
                InputTime = now,
                OutputTime = now + TimeSpan.FromMinutes(10),
            });
            Records.Add(new Record
            {
                IdCode = "fix0003",
            });
            Records.Add(new Record
            {
                IdCode = "fix0004",
                InputTime = now + TimeSpan.FromMinutes(5),
                OutputTime = now + TimeSpan.FromMinutes(5) + TimeSpan.FromMinutes(10),
            });
        }


  [1]: https://i.stack.imgur.com/stuCx.png