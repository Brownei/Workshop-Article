<p align="center">
  <a href="" rel="noopener">
 <img src="https://docs.reach.sh/assets/logo.png" alt="Project logo"></a>
</p>
<h3 align="center">RSVP</h3>

<div align="center">


</div>

---

<p align="center"> Workshop : RSVP
    <br> 
</p>

In this workshop we would be building a RSVP DApp. 
Assuming an organisation is planning to hold an event and they are struggling to get a large and dedicated audience; Our RSVP DApp would a very vital solution.

The DApp allows organizers to set up their event and set the price of their tickets. Then, their expected audience can purchase the tickets for the allocated price. When they arrive at the event they get refunded the amount that they spent when purchasing the ticket.

This DApp helps event organizers acquire a dedicated audience for their  events ensuring a good percentage of their audience is present at the event or else they pay for their absence.

This workshop assumes that you've completed at least the basic tutorials on the reach platform. If  you have not please click the [link](https://docs.reach.sh/tut/#tuts) and check them out first.

Next using the terminal command below we will create and access a new directory.
```bash
$ mkdir -p ~/reach/rsvp && cd ~/reach/rsvp
```
Next download the reach commandline tool using the command below:

```bash
curl https://docs.reach.sh/reach -o reach ; chmod +x reach
```
Once that is done to confirm  that reach has been successfully installed run the following command 

```bash 
$ ./reach version
```
The output of that command should look like this `reach 0.1.11`

Next initialize our reach application using the following command 

```bash
./reach init
```

This generates 2 files in this directory:

- `index.mjs`

- `index.rsh`

`index.mjs` is the file that contains the testing interface.

`index.rsh` is the smart contract(the file that contains the reach code). 

# DApp Logic

Let us get straight to the point and break down the logic of the application.

- Firstly the deployer who in this case is the event organizer interacts with the contract first by setting the parameters of the reservation then deploys the contract,  letting the expected audience know that they can start purchasing their tickets.

- Secondly the expected guest goes ahead and attaches to the contract then purchases the tickets by paying into the contract and then gets added to the list of the expected guests.

- Thirdly the expected guest interacts with the contract through a predefined API which prompts to indicate if he or she would be present at the event.

- Once the guest indicates that they will be present, through the organizers admin table, he/she will see the guest who indicated that they would be present and is then allowed to verify if they are actually present by checking them in at the event entrance. If the organizer verify that  they are present at the event he checks them in on his admin table which triggers the re-fund of their ticket expensed during the event reservation. At the end of the event, the absentees expenses are then transfered to the organizer and the contract ends.

Now let me show you how all these steps are been built out using reach.

# Reach Implementaion

Lets begin with defining the organizer and necessary API's for the guest and the organizer.

```js
"reach 0.1";

export const main = Reach.App(() => {
  const Event = Participant('Admin', {
    price: UInt,
    deadline: UInt,
    ready: Fun([], Null),
    informRSVP: Fun([Address], Null),
    informTimeout: Fun([], Null)
  });

  const Attendee = API('Attendee', {
    isGoingToBePresent: Fun([], Bool),
    seePrice: Fun([], UInt)
  });
  const Checkin = API('Checkin', {

    isPresent: Fun([Address], Bool),
    timesUp: Fun([], Bool),
  });
  const Status = API('Status', {

    seeStatus: Fun([], Bool),
  })
    
  init();
```
On the Event participant interface the following functions are present :

- `price` - this is used to represent the price of the reservation tickets.
- `deadline` -   this is used to represent the end of the event
- `ready` - this is used to tell the attachers that the event reservation is ready and they can start purchasing their tickets.
- `informRSVP` -  this is used to inform the organizer of the present guests.
- `informTimeout`  - this is a precautionary  funtion that is activated when the organizer is inactive for a fixed time period.

Next on the Attendee API:

- `isGoingToBePresent` - is an API function which the guests use to purchase the reservation ticket and indicate that they would be present at the event.
- `seePrice`  -  is the function  the  guest will use to see the price of the reservation ticket

Next the Checkin API:

- `isPresent` - is used by the organizer to check if the guests arrived at the event and then refunds the guests the amount they spent to purchase ticket.
- `timesup`- is used to signal the end of the event

And finally  Status API:

- `seeStatus`- is used by the guests to confirm that they have been checked in or not.

Next we would be going to the implementations of the DApp logic

```js
Event.only(() => {
    const price = declassify(interact.price);
    const deadline = declassify(interact.deadline);
  });
  Event.publish(price, deadline);
  commit();
  Event.publish();
  Event.interact.ready();
 
    
  const deadlineBlock = relativeTime(deadline);
  const RSVPs = new Set();
  const Present = new Set();
```

-  The organizer interacts with his functions and sets the price and deadline then publishes it to the blockchain.
-  Next he lets the attachers know that the reservation is now open.
- `deadlineBlock` - is used to format the deadline using `relativeTime` which is a time argument perfect for the deadline.
- Next we created 2 sets, a set is an array used to store addresses in Reach, the set `RSVPS` is used to store  the addresses of the guests that have purchased the tickets and `Present` is  a set used to store the addresses  of the present guests, i.e  the guests that attended the event.

```js
const [ keepGoing, howMany] =
    parallelReduce([true, 0])
    .invariant(balance() == howMany * price)
    .invariant(RSVPs.Map.size() == howMany)
    .while( keepGoing )
    .api_(Attendee.isGoingToBePresent, () => {
      check( ! RSVPs.member(this), "not yet" );
      return [ price, (k) => {
        k(true);
        RSVPs.insert(this);
        Event.interact.informRSVP(this);
        
        return [ keepGoing, howMany + 1 ];
      }];
    })
    .api_(Attendee.seePrice, () => {
      return [ 0, (k) => {
        k(price);
        
        return [ keepGoing, howMany  ];
      }];
    })
    .api_(Checkin.isPresent, (who) => {
      check( this == Event, "you are the boss");
      check( RSVPs.member(who), "yep" );
      return [ 0, (k) => {
        k(true);
        transfer(price).to(who);
        RSVPs.remove(who);
        Present.insert(who);
        
        return [ keepGoing, howMany - 1 ];
      }];
    })
    .api_(Status.seeStatus, () => {
      return [0, (k) => {
        k(Present.member(this))

        return [ keepGoing, howMany];
      }]
    })
    .timeout( deadlineBlock, () => {
      const [ [], k ] = call(Checkin.timesUp);
      k(true);
      Event.interact.informTimeout();
      return [ false, howMany ]
    });
```
The above snippet represents all the APIs  and their functions implimentation.
-  The APIs are called in a while loop embedded  inside a parralel reduce. To learn more about parralel reduce check out the Reach docs [link](https://docs.reach.sh/rsh/consensus/#term_parallel%20reduce%20statement)  
- The invariants are defined and the while loop can proceed. It starts with the `isGoingToBePresent` function and it stores the callers address in the set `RSVPS` and  allows the organizer to see that there has been a reservation.
- Next is the `seePrice` warning the guest of the cost of being absent.
- Next  is the `isPresent` which is used by the organizer at the entrance to check in the present guests by removing them from the reservation list and adding them to the present list and also refunding them back the price of the reservation.
- Next is the `seeStatus` which returns true or false if the guest has been verified as present or not.
- And finally is the `timeout` function used to signal the end of the event.

At the end of the event the leftovers in the contract are then transfered to the organizer, i.e the expenses of the absentees.

```js
  const leftovers = howMany;
  transfer(leftovers * price).to(Event);
  commit();
  exit();
  ```

Below is the complete `index.rsh` and the `index.mjs` file used to test the smart contract.

-  `index.rsh`
```js
"reach 0.1";

export const main = Reach.App(() => {
  const Event = Participant('Admin', {
    price: UInt,
    deadline: UInt,
    ready: Fun([], Null),
    informRSVP: Fun([Address], Null),
    informTimeout: Fun([], Null)
  });

  const Attendee = API('Attendee', {
    isGoingToBePresent: Fun([], Bool),
    seePrice: Fun([], UInt)
  });
  const Checkin = API('Checkin', {

    isPresent: Fun([Address], Bool),
    timesUp: Fun([], Bool),
  });
  const Status = API('Status', {

    seeStatus: Fun([], Bool),
  })
    
  init();

  Event.only(() => {
    const price = declassify(interact.price);
    const deadline = declassify(interact.deadline);
  });
  Event.publish(price, deadline);
  commit();
  Event.publish();
  Event.interact.ready();
 
    
  const deadlineBlock = relativeTime(deadline);
  const RSVPs = new Set();
  const Present = new Set();


  const [ keepGoing, howMany] =
    parallelReduce([true, 0])
    .invariant(balance() == howMany * price)
    .invariant(RSVPs.Map.size() == howMany)
    .while( keepGoing )
    .api_(Attendee.isGoingToBePresent, () => {
      check( ! RSVPs.member(this), "not yet" );
      return [ price, (k) => {
        k(true);
        RSVPs.insert(this);
        Event.interact.informRSVP(this);
        
        return [ keepGoing, howMany + 1 ];
      }];
    })
    .api_(Attendee.seePrice, () => {
      return [ 0, (k) => {
        k(price);
        
        return [ keepGoing, howMany  ];
      }];
    })
    .api_(Checkin.isPresent, (who) => {
      check( this == Event, "you are the boss");
      check( RSVPs.member(who), "yep" );
      return [ 0, (k) => {
        k(true);
        transfer(price).to(who);
        RSVPs.remove(who);
        Present.insert(who);
        
        return [ keepGoing, howMany - 1 ];
      }];
    })
    .api_(Status.seeStatus, () => {
      return [0, (k) => {
        k(Present.member(this))

        return [ keepGoing, howMany];
      }]
    })
    .timeout( deadlineBlock, () => {
      const [ [], k ] = call(Checkin.timesUp);
      k(true);
      Event.interact.informTimeout();
      return [ false, howMany ]
    });
    
  const leftovers = howMany;
  transfer(leftovers * price).to(Event);
  commit();
  exit();
});
```
- `index.mjs`

```js
import { loadStdlib } from '@reach-sh/stdlib';
import * as backend from './build/index.main.mjs';

const stdlib = loadStdlib({ REACH_NO_WARN: 'Y' });
const sbal = stdlib.parseCurrency(100);
const accD = await stdlib.newTestAccount(sbal);

const deadline = stdlib.connector === 'CFX' ? 500 : 250;

const ctcD = accD.contract(backend);
//console.log(ctcD);
await stdlib.withDisconnect(() =>
  ctcD.p.Admin({
    deadline,
    price: stdlib.parseCurrency(25),
    ready: stdlib.disconnect
  })
);

const users = await stdlib.newTestAccounts(10, sbal);

const willError = async (f, whoi) => {
  const who = users[whoi];  
  let e;
  try {
    await f();
    e = false;
  } catch (te) {
    e = te;
  }
  if ( e === false ) {
    throw Error(`Expected to error, but didn't`);
  }
  console.log('NO RSVP SLOT FOR', stdlib.formatAddress(who) );
};
const willErrorRepeat = async (f, whoi) => {
    const who = users[whoi];  
    let e;
    try {
      await f();
      e = false;
    } catch (te) {
      e = te;
    }
    if ( e === false ) {
      throw Error(`Expected to error, but didn't`);
    }
    console.log(stdlib.formatAddress(who), 'ALREADY HAS AN RSVP SPOT' );
  };
  const willErrorCheckin = async (f, whoi) => {
    const who = users[whoi];  
    let e;
    try {
      await f();
      e = false;
    } catch (te) {
      e = te;
    }
    if ( e === false ) {
      throw Error(`Expected to error, but didn't`);
    }
    console.log(stdlib.formatAddress(who), 'HAS ALREADY CHECKED IN' );
  };
const ctcWho = (whoi) =>
  users[whoi].contract(backend, ctcD.getInfo());
const rsvp = async (whoi) => {
  const who = users[whoi];
  const ctc = ctcWho(whoi);
  console.log('RSVP of', stdlib.formatAddress(who));
  await ctc.apis.Attendee.isGoingToBePresent();
};
const do_checkin = async (ctc, whoi) => {
  const who = users[whoi];
  console.log('Check in of', stdlib.formatAddress(who));
  await ctc.apis.Checkin.isPresent(who);
};
const checkin = async (whoi) => do_checkin(ctcD, whoi);
const timesup = async () => {
  console.log('I think time is up');
  await ctcD.apis.Checkin.timesUp();
};
const status = async (whoi) => {
  const who = users[whoi];
  const ctc = ctcWho(whoi);
  const y = await ctc.apis.Status.seeStatus();
  //console.log(y)
 if (y) console.log(stdlib.formatAddress(who), ' Succesfully Registered For The Event')
  
  else console.log(stdlib.formatAddress(who), 'Did Not Attend The Event')
}

await rsvp(0)
await rsvp(1) // Not checked in
await rsvp(2)
await rsvp(4)
await rsvp(6)
await rsvp(7) // Not checked in
await willErrorRepeat(() => rsvp(6),6) //attempt to rsvp twice
await checkin(4)
await willError(() => checkin(3),3) // not matching rsvp found
await willError(() => checkin(5),5) // no matching rsvp found
await checkin(0)
await checkin(2)
await checkin(6)
await willErrorCheckin(() => checkin(6),6) // attempt to checkin twice
await status(0)
await status(1)
await status(2)
await status(3)
await status(4)
await status(5)
await status(6)
await status(7)
await status(8)

console.log(`We're gonna wait for the deadline`);
await stdlib.wait(deadline);

await timesup();

for ( const who of [ accD, ...users ]) {
  console.warn(stdlib.formatAddress(who), 'has',
    stdlib.formatCurrency(await stdlib.balanceOf(who)));
}
```
## Discussion
If you made it down here congratulations on finishing this workshop. You've sucsessfully implenemted an RSVP Ticketing Decentralized Application on the blockchain.

You can  view the full codebase at [my repo](https://github.com/Brownei/Reach-RSVP)
If you found this workshop rewarding please let us know on [the Discord Community](https://discord.gg/AZsgcXu).

Thank You !!

