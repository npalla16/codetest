
  
The Live Order Board
In this exercise, the Live Order Board is to tell the users how much demand for silver bars there is on the market.


Features
Register an order
The LiveOrderBoard should allow the user to register an Order, which, according to the requirements, "must contain the following fields": user id, order quantity (e.g.: 3.5 kg), price per kg (e.g.: £306/+) as well as the type of order: either buy or sell.

Modelling the above using primitive numeric data types would negatively affect the precision of our calculations, while relying on objects such as String or BigDecimal would prevent us from retaining the semantic meaning of the domain concepts.

To address those problems and avoid re-inventing the wheel where good open-source libraries exist, information contained in an Order is modelled as follows:

user id - to retain the meaning of a domain concept, the user id is modelled using a 'Tiny Type' mini-pattern.
order quantity (e.g.: 3.5 kg) - to make the basic arithmetic operations easier and retain the meaning of the domain concept the quantity of the order is modelled using the unitsofmeasurement library, which is a reference implementation of the JSR 363: Units of Measurement API
price per kg (e.g.: £306) - to allow for precise comparison of orders, the price is modelled using the Joda Money library, which Money type is wrapped in the PricePerKg to retain the domain concept
order type: BUY or SELL - the type of the order is modelled using a nested enumeration, defined as part of the Order class
To see how this design could be improved to better follow OOP, see my notes in the Improving the design section below.

With the above in place, the registration of an Order looks as follows:

LiveOrderBoard board = new LiveOrderBoard();

board.register(new Order(
        new UserId("user 1"),
        Quantities.getQuantity(3.5, KILOGRAM),
        new PricePerKg(Money.of(GBP, 306)),
        Order.Type.Buy
));
However, the instantiation of an Order can be made more readable thanks to static factory methods:

LiveOrderBoard board = new LiveOrderBoard();

board.register(new Order(
        UserId.of("user 1"),
        Quantities.getQuantity(3.5, KILOGRAM),
        PricePerKg.of(GBP, 306),
        Order.Type.Buy
));
To simplify this further, I've created a simple DSL which you can see being used in the automated tests:

board.register(buy(kg(3.5), £(306), Alice));
This tiny DSL helps to highlight the important details of the tests and avoid most of the boilerplate code.

Side note 1: The £(amount) function is an alias for PricePerKg.of(GBP, amount), which makes the test easier to read, but to be useful requires the developers working with such code to be able to use the £ symbol on their keyboards. In an international setting, calling the function something like gbp(amount) would be more keyboard-friendly.

Side note 2: The order quantity could've also been designed as a Tiny Type and use unitsofmeasurement under the hood. This however felt like introducing unnecessary indirection as Quantity<Mass> coming from unitsofmeasurement seems descriptive enough.

Cancel a registered order
It seemed sensible that an Order can be cancelled only by the user who placed it.

LiveOrderBoard board = new LiveOrderBoard();

Order order = buy(kg(3.5), £(306), U);

board.register(order);
board.cancel(order);
To keep it simple, the current implementation does not throw an exception when a user tries to cancel someone else's order (nor it will notify the authorities ;) ) - the operation will just not affect the state of the board.

Summary information of live orders
The LiveOrderBoard is backed by a simple List<Order> implementation, which makes the cost of inserting (registering) and removing (cancelling) Orders linear - O(n).

To generate a "live summary", the orders in the list need to be aggregated and mapped to instances of OrderSummary. As this is done using Java 8 Streams API, the cost of aggregation and mapping is kept at O(n), as the list only needs to be processed once.

The list of OrderSummary objects is sorted using the OrderComparator, which is a composition of three other comparators. The comparators in question fulfil the ordering requirements of the exercise, but are hard-coded in the OrderComparator. If there was a need to make the ordering rules more sophisticated the list of comparators could be made configurable, or an alternative comparator injected into the LiveOrderBoard.

Design Considerations
Scalability
The solution of the exercise is implemented to optimise for readability and low cost of maintenance of the code base. The time complexity of LiveOrderBoard access operations is fairly good at O(n), and all the domain objects are immutable and thread-safe.

However, the List<Order> backing the LiveOrderBoard would not be sufficient to support a real-world implementation of a metal market exchange, which presumably would be required to process large order volumes in parallel.

Improving the design
The exercise puts a constraint on the design of the Order class saying that:

Order must contain these fields:

user id
order quantity (e.g.: 3.5 kg)
price per kg (e.g.: £303)
order type: BUY or SELL
If the requirement stated that the Order class must contain the following information rather than fields, an alternative, more OOP-friendly design would be possible.

Instead of defining an Order as

new Order(
        UserId.of("user 1"),
        Quantities.getQuantity(10, KILOGRAM),
        PricePerKg.of(GBP, 203),
        Order.Type.Buy
);
we could introduce a Bid interface, with two concrete implementations: Buy and Sell, so that:

new Order(
        UserId.of("user 1"),
        Quantities.getQuantity(10, KILOGRAM),
        new Buy(PricePerKg.of(GBP, 203)
);
The same approach would work with the OrderSummary as it also needs to know about the type of the order.

Instead of:

new OrderSummary(Quantities.getQuantity(10, KILOGRAM), PricePerKg.of(GBP, 203), Order.Type.Buy);
we'd have:

new OrderSummary(Quantities.getQuantity(10, KILOGRAM), new Buy(PricePerKg.of(GBP, 203)));
The above refactoring would allow us to use polymorphism instead of if statements in the OrderComparator and make the Bid a part of both the Order and OrderSummary, rather than a utility class used merely to make the conversion from Order to OrderSummary easier.