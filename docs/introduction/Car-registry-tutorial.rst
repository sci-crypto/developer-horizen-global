====================================
Car Registry Tutorial
====================================

Car Registry App High-Level Overview
####################################

The `Lambo-Registry app <https://github.com/HorizenOfficial/lambo-registry>`_is a demo dApp implemented as a sidechain, that makes use of custom data and logic. It was developed to serve as a practical example of how the SDK can be extended.
From a functional point of view, the application acts as a repository of existing cars and their owners, and offers to its users the possibility to sell and buy cars. It is a demo application, so it does not include all the needed checks and functionalities that a production application would need; for instance, users are now able to register a car by broadcasting a simple "Car Declaration" trasaction. We could think that, in a real-world scenario, the ability to declare the existence of a new car in the sidechain, might be instead subject to the inclusion in the transaction of a certificate signed by the Department of Motor Vehicles, that guarantees that the car exists and it’s owned by a user with a specified public key.

To sum up, the Lambo-Registry applications just accepts transactions that create cars, and then provides the following functionalities:

    1. It stores information that identifies a specific car, such as vehicle identification number (VIN), model, production year, color.
    2. It allows car owners to be able to prove their ownership of the cars anonymously.
    3. It gives the possibility to sell a car in exchange for ZEN. 



User stories:
#############

As usual, the first step of software development is the analysis. Let's list the functional requirements of our dApp as some simple user requests ("R"), and then the associated design decisions ("D"):

**R: I want to add my car to the Car Registry App.**

*D:* We'll introduce a transaction that creates a "Car Entry Box", with all the vehicle's identification information (VIN, manufacturer, model, year, registration number). The proposition associated to this box is the public key of the owner of the car. When a Car Box is created, the sidechain should verify that the vehicle identification information are unique to this sidechain.

**R: I want to sell my car.**

*D:* We'll introduce a "Car Sell Order Box" that includes the vehicle's information and its price in ZEN. Cars can exist in the sidechain either as a "Car Entry Box" or as a "Car Sell Order Box", but not both at the same time. The Car Sell Order Box will contain also the public key of the prospective buyer, so we assume that some kind of negotiation/agreement between the seller and the buyer took place off-chain. When a sell order is created, the sidechain will have to verify that there is no other active sell order for the same vehicle.


**R: I want to buy a car.**

*D:* To buy a car, the user will have to create a new transaction that accepts a sell order. That sell order must specify the user's public key. The transaction will create a new Car Entry Box, closed by the new owner's public key as proposition. The transaction will also transfer the correct amount of ZEN coins from the buyer to the seller.

**R: I've changed my mind, and don't want to sell my car any more.**

*D:* If the sell order is still active, it can be recalled by its creator. The car owner will create a new transaction containing the Car Sell Order as input, and a Car Entry Box closed by his public key as output.

**R: I want to see all the cars I own, and the ones that have been offered to me.**

*D:* This piece of information will be managed by ApplicationWallet. We can use the SDK standard endpoint "wallet/allBlocks" and filter by box type.


We can now start the development process, by addressing the data representation.


## Boxes

When designing a new application, the first thing to do is identify the needed custom boxes, and their properties. Boxes are the basic objects that describe the state of our application. The Lambo-registry example implements the following custom boxes:

- CarBox
  A Box that represents a car instance. The following properties were selected to describe a car:

  - vehicle identification number (vin)
  - year of pruduction
  - model
  - color
  
- CarSellOrderBox
  A Box that represents the intention to sell a car to someone. It has the same properties of a car, a price (in ZEN), and it is closed by a special proposition which can be opened either by the seller (to remove the car from sale) or the buyer (to complete the purchase).

Let's have a closer look at the code that defines a CarBox:

```
    @JsonView(Views.Default.class)
    @JsonIgnoreProperties({"carId", "value"})
    public final class CarBox extends AbstractNoncedBox<PublicKey25519Proposition, CarBoxData, CarBox> {

        public CarBox(CarBoxData boxData, long nonce) {
            super(boxData, nonce);
        }

        @Override
        public byte boxTypeId() {
            return CarBoxId.id();
        }

        @Override
        public BoxSerializer serializer() {
            return CarBoxSerializer.getSerializer();
        }

        @Override
        public byte[] bytes() {
            return Bytes.concat(
                    Longs.toByteArray(nonce),
                    CarBoxDataSerializer.getSerializer().toBytes(boxData)
            );
        }

        public static CarBox parseBytes(byte[] bytes) {
            long nonce = Longs.fromByteArray(Arrays.copyOf(bytes, Longs.BYTES));
            CarBoxData boxData = CarBoxDataSerializer.getSerializer().parseBytes(Arrays.copyOfRange(bytes, Longs.BYTES, bytes.length));
            return new CarBox(boxData, nonce);
        }

        public String getVin() {
            return boxData.getVin();
        }

        public int getYear() {
            return boxData.getYear();
        }

        public String getModel() {
            return boxData.getModel();
        }

        public String getColor() {
            return boxData.getColor();
        }

        public byte[] getCarId() {
            return Bytes.concat(
                    getVin().getBytes(),
                    Ints.toByteArray(getYear()),
                    getModel().getBytes(),
                    getColor().getBytes()
            );
        }
    }
```

Let's start from the top declaration:


```
    @JsonView(Views.Default.class)
    @JsonIgnoreProperties({"carId", "value"})
    public final class CarBox extends AbstractNoncedBox<PublicKey25519Proposition, CarBoxData, CarBox> {
 ```   

 Our class extends the *AbstractNoncedBox* default class, is locked by a standard *PublicKey25519Proposition* and keeps all its properties into an object of type CarBoxData.
 The annotation @JsonView instructs the SDK to use a default viewer to convert an instance of this class into JSON format when a CarBox is included in the result of an http API endpoint. With that, there is no need to write the conversion code: all the properties associated to getter methods of the class are automatically converted to json attributes. 
 For example, since our class has a getter method "getModel()", the json will contain the attribute "model" with its value. 
 We can specify some properties that must be excluded from the json output with the @JsonIgnoreProperties annotation.

 The constructor of boxes extending AbstractNoncedBox is very simple, it just calls the superclass with two parameters: the BoxData and the nonce.

 ```
    public CarBox(CarBoxData boxData, long nonce) {
        super(boxData, nonce);
    }
```    
The BoxData is a container of all the properties of our Box, we'll have a look at it later.
The nonce is a random number that allows the generation of different hash values also if the inner properties of two boxes have the same values.


 ```
    @Override
    public byte boxTypeId() {
        return CarBoxId.id();
    }
 ```   

The method boxTypeId() returns the id of this box type: every custom box needs to have a unique type id inside the application. Note that the ids of custom boxes can overlap with the ids of the standard boxes (e.g. you can re-use the id type 1 that is already used for standard coin boxes).

The next three methods are used for serialization and de-serialization of our Box: they define the serializer to be used, and the methods used to generate a byte array from the box and to obtain the box back from the byte array (note that they delegate the byte handling logic to the CarBoxData):


 ```
    @Override
    public BoxSerializer serializer() {
        return CarBoxSerializer.getSerializer();
    }

    @Override
    public byte[] bytes() {
        return Bytes.concat(
                Longs.toByteArray(nonce),
                CarBoxDataSerializer.getSerializer().toBytes(boxData)
        );
    }

    public static CarBox parseBytes(byte[] bytes) {
        long nonce = Longs.fromByteArray(Arrays.copyOf(bytes, Longs.BYTES));
        CarBoxData boxData = CarBoxDataSerializer.getSerializer().parseBytes(Arrays.copyOfRange(bytes, Longs.BYTES, bytes.length));
        return new CarBox(boxData, nonce);
    }
 ```

 The last methods of the class are just the getters of the box properties. In particular getCarId() is an example of a property that is the result of operations performed on other stored properties.

 There are three more classes related to our CarBox: the boxdata and the serializers. Let's have a closer look at them.

 ### BoxData

 BoxData allow us to group all the box properties and their serialization and de-serialization logic in a single container object. Altought its use is not mandatory (you can define field properties directly inside the Box), it is required if you choose to extend the base class AbstractNoncedBox, as we did for the CarBox, and it is in any case a good practice.

 
```
@JsonView(Views.Default.class)
public final class CarBoxData extends AbstractNoncedBoxData<PublicKey25519Proposition, CarBox, CarBoxData> {

    private final String vin;   // Vehicle Identification Number
    private final int year;     // Car manufacture year
    private final String model; // Car Model
    private final String color; // Car color

    public CarBoxData(PublicKey25519Proposition proposition, String vin,
                      int year, String model, String color) {
        super(proposition, 0);
        this.vin = vin;
        this.year = year;
        this.model = model;
        this.color = color;
    }


    @Override
    public CarBox getBox(long nonce) {
        return new CarBox(this, nonce);
    }

    @Override
    public byte[] customFieldsHash() {
        return Blake2b256.hash(
                Bytes.concat(
                        vin.getBytes(),
                        Ints.toByteArray(year),
                        model.getBytes(),
                        color.getBytes()));
    }

    @Override
    public NoncedBoxDataSerializer serializer() {
        return CarBoxDataSerializer.getSerializer();
    }

    @Override
    public byte boxDataTypeId() {
        return CarBoxDataId.id();
    }

    @Override
    public byte[] bytes() {
        return Bytes.concat(
                proposition().bytes(),
                Ints.toByteArray(vin.getBytes().length),
                vin.getBytes(),
                Ints.toByteArray(year),
                Ints.toByteArray(model.getBytes().length),
                model.getBytes(),
                Ints.toByteArray(color.getBytes().length),
                color.getBytes()
        );
    }

    public static CarBoxData parseBytes(byte[] bytes) {
        int offset = 0;

        PublicKey25519Proposition proposition = PublicKey25519PropositionSerializer.getSerializer()
                .parseBytes(Arrays.copyOf(bytes, PublicKey25519Proposition.getLength()));
        offset += PublicKey25519Proposition.getLength();

        int size = Ints.fromByteArray(Arrays.copyOfRange(bytes, offset, offset + Ints.BYTES));
        offset += Ints.BYTES;

        String vin = new String(Arrays.copyOfRange(bytes, offset, offset + size));
        offset += size;

        int year = Ints.fromByteArray(Arrays.copyOfRange(bytes, offset, offset + Ints.BYTES));
        offset += Ints.BYTES;

        size = Ints.fromByteArray(Arrays.copyOfRange(bytes, offset, offset + Ints.BYTES));
        offset += Ints.BYTES;

        String model = new String(Arrays.copyOfRange(bytes, offset, offset + size));
        offset += size;

        size = Ints.fromByteArray(Arrays.copyOfRange(bytes, offset, offset + Ints.BYTES));
        offset += Ints.BYTES;

        String color = new String(Arrays.copyOfRange(bytes, offset, offset + size));

        return new CarBoxData(proposition, vin, year, model, color);
    }

    public String getVin() {
        return vin;
    }

    public int getYear() {
        return year;
    }

    public String getModel() {
        return model;
    }

    public String getColor() {
        return color;
    }

    @Override
    public String toString() {
        return "CarBoxData{" +
                "vin=" + vin +
                ", proposition=" + proposition() +
                ", model=" + model +
                ", color=" + color +
                ", year=" + year +
                '}';
    }
}

 ```

Let's look in detail at the code above, starting from the beginning:

 
 ```
    @JsonView(Views.Default.class)
    public final class CarBoxData extends AbstractNoncedBoxData<PublicKey25519Proposition, CarBox, CarBoxData> {
    
 ```

Also this time, we have a basic class we can extend: AbstractNoncedBoxData.

 ```
  public CarBoxData(PublicKey25519Proposition proposition, String vin,
                     int year, String model, String color) {
       super(proposition, 0);
       this.vin = vin;
       this.year = year;
       this.model = model;
       this.color = color;
   }
 ```

The constructor receives all the box properties, and the proposition that locks it. The proposition is passed up to the super-class constructor, which  receives also a long number representing the ZEN value of the box. For boxes that don't handle coins (like this one) we can just pass a 0 constant value.

 ```
   @Override
   public CarBox getBox(long nonce) {
       return new CarBox(this, nonce);
   }
  ```

  The getBox(long nonce) is a helper method used to generate a new box from the content of this boxdata.

 ```
  @Override
   public byte[] customFieldsHash() {
       return Blake2b256.hash(
               Bytes.concat(
                       vin.getBytes(),
                       Ints.toByteArray(year),
                       model.getBytes(),
                       color.getBytes()));
   }
 ```
The method customFieldsHash() is used by the sidechain to generate a unique hash for each box intance: it needs to be defined in a way such that different property values of a boxdata always produce a different hash value. To achieve this, the code uses a scorex helper class (scorex.crypto.hash.Blake2b256) that generates a hash from a bytearray; the bytearray is the concatenation of all the properties values.

Boxdata, as Box, has some methods to define its serializer, and a unique type id:

 ```
    @Override
   public NoncedBoxDataSerializer serializer() {
       return CarBoxDataSerializer.getSerializer();
   }

   @Override
   public byte boxDataTypeId() {
       return CarBoxDataId.id();
   }
 ```

 Two very important methods are bytes() and parseBytes(): they contain the logic to serialize and deserialize properties and proposition. The code is quite verbose but simple: bytes() returns a byte array that is the concatenation of all the properties values, while parseBytes() reads it and write the values back. Note that for variable-lenght fields like strings, the field lenght needs to be first known and serialized, and made part of the bytearray, so that parseBytes() can then read the correct lenght of bytes of that field. You can see it in the code that serializes the car model string:

   ```
        return Bytes.concat(
                ....
                Ints.toByteArray(model.getBytes().length),
                model.getBytes(),
                ....
        );
 ```

 and this is the code in parseBytes() that reads the bytearray and writes back the car model:

```
        size = Ints.fromByteArray(Arrays.copyOfRange(bytes, offset, offset + Ints.BYTES));
        offset += Ints.BYTES;
        String model = new String(Arrays.copyOfRange(bytes, offset, offset + size));
 ```


 As expected, the class include all the getters of every custom property (getModel(), getColor() etc..). Also, the toString() method is redefined to print out the content of boxdata in a more user-friendly format:

  ```
    @Override
    public String toString() {
        return "CarBoxData{" +
                "vin=" + vin +
                ", proposition=" + proposition() +
                ", model=" + model +
                ", color=" + color +
                ", year=" + year +
                '}';
    }
 ```
  

 ### BoxSerializer and BoxDataSerializer

 Serializers are companion classes that are invoked by the SDK every time a Scorex reader and writer needs to deserialize or serialize a Box. We define one serializers/deserializers both for box and for boxdata.
 As you can see in the code below, since the "heavy" byte handling happens inside boxdata, their logic is very simple: they just call the right methods already defined in the associated (Box or BoxData) objects.

 ```
 public final class CarBoxSerializer implements BoxSerializer<CarBox> {

    private static final CarBoxSerializer serializer = new CarBoxSerializer();

    private CarBoxSerializer() {
        super();
    }

    public static CarBoxSerializer getSerializer() {
        return serializer;
    }

    @Override
    public void serialize(CarBox box, Writer writer) {
        writer.putBytes(box.bytes());
    }

    @Override
    public CarBox parse(Reader reader) {
        return CarBox.parseBytes(reader.getBytes(reader.remaining()));
    }
}
 ```

 ```
    public final class CarBoxDataSerializer implements NoncedBoxDataSerializer<CarBoxData> {

        private static final CarBoxDataSerializer serializer = new CarBoxDataSerializer();

        private CarBoxDataSerializer() {
            super();
        }

        public static CarBoxDataSerializer getSerializer() {
            return serializer;
        }

        @Override
        public void serialize(CarBoxData boxData, Writer writer) {
            writer.putBytes(boxData.bytes());
        }

        @Override
        public CarBoxData parse(Reader reader) {
            return CarBoxData.parseBytes(reader.getBytes(reader.remaining()));
        }
    }
 ```


## Transactions

If Boxes are the objects that describe the state of our application, transactions are the actions that can the application state. They typically do that by opening (and therefore removing) some boxes ("input"), and creating new ones ("output").

Our Car Registry application definies the following custom transactions:
- CarDeclarationTransaction
  a transaction that declares a new car (by creating a new CarBox).
- SellCarTransaction
  it creates a sell order for a car: a CarBox is "spent", and a CarSellOrderBox containing all the data of the car to be sold is created.
- BuyCarTransaction
  this transaction is used either by the buyer to accept the sell order, or by the seller to cancel it. It opens a CarSellOrderBox, and creates a CarBox (if it's a sell order cancellation, the new CarBox will be assigned to the original owner).

Let's look at the code of the last one, BuyCarTransaction, that is slightly more complicated than the other two:


```
public final class BuyCarTransaction extends AbstractRegularTransaction {


    private final CarBuyOrderInfo carBuyOrderInfo;
    private List<NoncedBox<Proposition>> newBoxes;

    public BuyCarTransaction(List<byte[]> inputRegularBoxIds,
                             List<Signature25519> inputRegularBoxProofs,
                             List<RegularBoxData> outputRegularBoxesData,
                             CarBuyOrderInfo carBuyOrderInfo,
                             long fee,
                             long timestamp) {
        super(inputRegularBoxIds, 
              inputRegularBoxProofs, 
              outputRegularBoxesData, 
              fee, timestamp);
        this.carBuyOrderInfo = carBuyOrderInfo;
    }

    @Override
    public List<BoxUnlocker<Proposition>> unlockers() {
        // Get Regular unlockers from base class.
        List<BoxUnlocker<Proposition>> unlockers = super.unlockers();

        BoxUnlocker<Proposition> unlocker = new BoxUnlocker<Proposition>() {
            @Override
            public byte[] closedBoxId() {
                return carBuyOrderInfo.getCarSellOrderBoxToOpen().id();
            }

            @Override
            public Proof boxKey() {
                return carBuyOrderInfo.getCarSellOrderSpendingProof();
            }
        };
        unlockers.add(unlocker);
        return unlockers;
    }


    @Override
    public List<NoncedBox<Proposition>> newBoxes() {
        if(newBoxes == null) {
            // Get new boxes from base class.
            newBoxes = new ArrayList<>(super.newBoxes());

            // Set CarBox with specific owner depends on proof. See CarBuyOrderInfo.getNewOwnerCarBoxData() definition.
            long nonce = getNewBoxNonce(carBuyOrderInfo.getNewOwnerCarBoxData().proposition(), newBoxes.size());
            newBoxes.add((NoncedBox) new CarBox(carBuyOrderInfo.getNewOwnerCarBoxData(), nonce));

            // If Sell Order was opened by the buyer -> add payment box for Car previous owner.
            if (!carBuyOrderInfo.isSpentByOwner()) {
                RegularBoxData paymentBoxData = carBuyOrderInfo.getPaymentBoxData();
                nonce = getNewBoxNonce(paymentBoxData.proposition(), newBoxes.size());
                newBoxes.add((NoncedBox) new RegularBox(paymentBoxData, nonce));
            }
        }
        return Collections.unmodifiableList(newBoxes);
    }

    // Specify the unique custom transaction id.
    @Override
    public byte transactionTypeId() {
        return BuyCarTransactionId.id();
    }

    // Define object serialization, that should serialize both parent class entries and CarBuyOrderInfo as well
    @Override
    public byte[] bytes() {
        ByteArrayOutputStream inputsIdsStream = new ByteArrayOutputStream();
        for(byte[] id: inputRegularBoxIds)
            inputsIdsStream.write(id, 0, id.length);

        byte[] inputRegularBoxIdsBytes = inputsIdsStream.toByteArray();
        byte[] inputRegularBoxProofsBytes = regularBoxProofsSerializer.toBytes(inputRegularBoxProofs);
        byte[] outputRegularBoxesDataBytes = regularBoxDataListSerializer.toBytes(outputRegularBoxesData);
        byte[] carBuyOrderInfoBytes = carBuyOrderInfo.bytes();

        return Bytes.concat(
                Longs.toByteArray(fee()),                               // 8 bytes
                Longs.toByteArray(timestamp()),                         // 8 bytes
                Ints.toByteArray(inputRegularBoxIdsBytes.length),       // 4 bytes
                inputRegularBoxIdsBytes,                                // depends on previous value (>=4 bytes)
                Ints.toByteArray(inputRegularBoxProofsBytes.length),    // 4 bytes
                inputRegularBoxProofsBytes,                             // depends on previous value (>=4 bytes)
                Ints.toByteArray(outputRegularBoxesDataBytes.length),   // 4 bytes
                outputRegularBoxesDataBytes,                            // depends on previous value (>=4 bytes)
                Ints.toByteArray(carBuyOrderInfoBytes.length),          // 4 bytes
                carBuyOrderInfoBytes                                    // depends on previous value (>=4 bytes)
        );
    }

    // Define object deserialization similar to 'toBytes()' representation.
    public static BuyCarTransaction parseBytes(byte[] bytes) {
        int offset = 0;

        long fee = BytesUtils.getLong(bytes, offset);
        offset += 8;

        long timestamp = BytesUtils.getLong(bytes, offset);
        offset += 8;

        int batchSize = BytesUtils.getInt(bytes, offset);
        offset += 4;

        ArrayList<byte[]> inputRegularBoxIds = new ArrayList<>();
        int idLength = NodeViewModifier$.MODULE$.ModifierIdSize();
        while(batchSize > 0) {
            inputRegularBoxIds.add(Arrays.copyOfRange(bytes, offset, offset + idLength));
            offset += idLength;
            batchSize -= idLength;
        }

        batchSize = BytesUtils.getInt(bytes, offset);
        offset += 4;

        List<Signature25519> inputRegularBoxProofs = regularBoxProofsSerializer.parseBytes(Arrays.copyOfRange(bytes, offset, offset + batchSize));
        offset += batchSize;

        batchSize = BytesUtils.getInt(bytes, offset);
        offset += 4;

        List<RegularBoxData> outputRegularBoxesData = regularBoxDataListSerializer.parseBytes(Arrays.copyOfRange(bytes, offset, offset + batchSize));
        offset += batchSize;

        batchSize = BytesUtils.getInt(bytes, offset);
        offset += 4;

        CarBuyOrderInfo carBuyOrderInfo = CarBuyOrderInfo.parseBytes(Arrays.copyOfRange(bytes, offset, offset + batchSize));
        return new BuyCarTransaction(inputRegularBoxIds, inputRegularBoxProofs, outputRegularBoxesData, carBuyOrderInfo, fee, timestamp);
    }

    // Set specific Serializer for BuyCarTransaction class.
    @Override
    public TransactionSerializer serializer() {
        return BuyCarTransactionSerializer.getSerializer();
    }
}

```

Let's start from the top declaration: 


```
    public final class BuyCarTransaction extends AbstractRegularTransaction {
 ```   
 Our class extends the *AbstractRegularTransaction* default class, an abstract class designed to handle regular coin boxes. Since blockchain transactions usually require the payment of a fee (including the three custom transactions of our Car Registry application), and to pay a fee you need to handle coin boxes, usually custom transactions will extend this abstract class.

 ```
    public BuyCarTransaction(List<byte[]> inputRegularBoxIds,
                             List<Signature25519> inputRegularBoxProofs,
                             List<RegularBoxData> outputRegularBoxesData,
                             CarBuyOrderInfo carBuyOrderInfo,
                             long fee,
                             long timestamp) {
        super(inputRegularBoxIds, 
              inputRegularBoxProofs, 
              outputRegularBoxesData, 
              fee, timestamp);
        this.carBuyOrderInfo = carBuyOrderInfo;
    }
```    
The constructor receives all the parameters related to regular boxes handling (box ids to be opened, proofs to open them, regular boxes to be created, fee to be paid and timestamp), and pass them up to the super-class. Moreover, it receives all other parameters specifically related to the custom boxes; in our example, the transaction needs info about the sell order that it needs to open, and it finds in the CarBuyOrderInfo object.


 ```
    @Override
    public List<BoxUnlocker<Proposition>> unlockers() {
        // Get Regular unlockers from base class.
        List<BoxUnlocker<Proposition>> unlockers = super.unlockers();

        BoxUnlocker<Proposition> unlocker = new BoxUnlocker<Proposition>() {
            @Override
            public byte[] closedBoxId() {
                return carBuyOrderInfo.getCarSellOrderBoxToOpen().id();
            }

            @Override
            public Proof boxKey() {
                return carBuyOrderInfo.getCarSellOrderSpendingProof();
            }
        };
        unlockers.add(unlocker);
        return unlockers;
    }

 ```   

The unlockers() method must return a list of BoxUnlocker's, that cointains the boxes which will be opened by this transaction, and the proofs to open them. The list returned from the superclass (in the first line of the method) contains the unlockers for the coin boxes, and it is combined with the unlocker for the CarSellOrderBox. As you can see we have used an inline declaration for the new unlocker, since it is a very simple object that has only two methods, one returning the box id to open and the other one the proof to open it.


 ```
    @Override
    public byte transactionTypeId() {
        return BuyCarTransactionId.id();
    }
 ```

Just like with boxes, also each transaction type must have a unique id, returned by the method transactionTypeId().

The last three methods of the class are related to the serialization handling.
The approach is very similar to what we saw for boxes: the methods bytes() and parseBytes(byte[] bytes) perform a "two-way conversion" into and from an array of bytes, while the serializer() method returns the serializer helper to operate with Scorex reader's and writer's.

As we did with the CarBox, also here we have choosen to code the low level "byte handling" logic inside the two methods bytes() and ParseBytes(byte[] bytes), keeping a very simple implementation for the serializer:

 ```
public final class BuyCarTransactionSerializer implements TransactionSerializer<BuyCarTransaction> {

    private static final BuyCarTransactionSerializer serializer = new BuyCarTransactionSerializer();

    private BuyCarTransactionSerializer() {
        super();
    }

    public static BuyCarTransactionSerializer getSerializer() {
        return serializer;
    }

    @Override
    public void serialize(BuyCarTransaction transaction, Writer writer) {
        writer.putBytes(transaction.bytes());
    }

    @Override
    public BuyCarTransaction parse(Reader reader) {
        return BuyCarTransaction.parseBytes(reader.getBytes(reader.remaining()));
    }
}
 ```

One of the parameters of the class constructor is CarBuyOrderInfo, an object that contains the needed info about the sell order we are handling. Let's take a look at its implementation:

 ```
public final class CarBuyOrderInfo {
    private final CarSellOrderBox carSellOrderBoxToOpen;  // Sell order box to be spent in BuyCarTransaction
    private final SellOrderSpendingProof proof;           // Proof to unlock the box above

    public CarBuyOrderInfo(CarSellOrderBox carSellOrderBoxToOpen, SellOrderSpendingProof proof) {
        this.carSellOrderBoxToOpen = carSellOrderBoxToOpen;
        this.proof = proof;
    }

    public CarSellOrderBox getCarSellOrderBoxToOpen() {
        return carSellOrderBoxToOpen;
    }

    public SellOrderSpendingProof getCarSellOrderSpendingProof() {
        return proof;
    }

    // Recreates output CarBoxData with the same attributes specified in CarSellOrder.
    // Specifies the new owner depends on proof provided:
    // 1) if the proof is from the seller then the owner remain the same
    // 2) if the proof is from the buyer then it will become the new owner
    public CarBoxData getNewOwnerCarBoxData() {
        PublicKey25519Proposition proposition;
        if(proof.isSeller()) {
            proposition = new PublicKey25519Proposition(carSellOrderBoxToOpen.proposition().getOwnerPublicKeyBytes());
        } else {
            proposition = new PublicKey25519Proposition(carSellOrderBoxToOpen.proposition().getBuyerPublicKeyBytes());
        }

        return new CarBoxData(
                proposition,
                carSellOrderBoxToOpen.getVin(),
                carSellOrderBoxToOpen.getYear(),
                carSellOrderBoxToOpen.getModel(),
                carSellOrderBoxToOpen.getColor()
        );
    }

    // Check if proof is provided by Sell order owner.
    public boolean isSpentByOwner() {
        return proof.isSeller();
    }

    // Coins to be paid to the owner of Sell order in case if Buyer spent the Sell order.
    public RegularBoxData getPaymentBoxData() {
        return new RegularBoxData(
                new PublicKey25519Proposition(carSellOrderBoxToOpen.proposition().getOwnerPublicKeyBytes()),
                carSellOrderBoxToOpen.getPrice()
        );
    }

    // CarBuyOrderInfo minimal bytes representation.
    public byte[] bytes() {
        byte[] carSellOrderBoxToOpenBytes = CarSellOrderBoxSerializer.getSerializer().toBytes(carSellOrderBoxToOpen);
        byte[] proofBytes = SellOrderSpendingProofSerializer.getSerializer().toBytes(proof);

        return Bytes.concat(
                Ints.toByteArray(carSellOrderBoxToOpenBytes.length),
                carSellOrderBoxToOpenBytes,
                Ints.toByteArray(proofBytes.length),
                proofBytes
        );
    }

    // Define object deserialization similar to 'toBytes()' representation.
    public static CarBuyOrderInfo parseBytes(byte[] bytes) {
        int offset = 0;

        int batchSize = BytesUtils.getInt(bytes, offset);
        offset += 4;

        CarSellOrderBox carSellOrderBoxToOpen = CarSellOrderBoxSerializer.getSerializer().parseBytes(Arrays.copyOfRange(bytes, offset, offset + batchSize));
        offset += batchSize;

        batchSize = BytesUtils.getInt(bytes, offset);
        offset += 4;

        SellOrderSpendingProof proof = SellOrderSpendingProofSerializer.getSerializer().parseBytes(Arrays.copyOfRange(bytes, offset, offset + batchSize));

        return new CarBuyOrderInfo(carSellOrderBoxToOpen, proof);
    }
}
 ```

 If you look at the code above, you can see that this object is not much more than a container of the information that needs to be processed: the CarSellOrderBox that should be opened, and the proof to open it. It then includes their getters, and a couple of "utility" methods: getNewOwnerCarBoxData() and getPaymentBoxData(). The first one, "getNewOwnerCarBoxData", creates a new CarBox with the same properties of the sold car, and "assigns" it (by locking it with the right proposition) to either the buyer or the seller, depending on who openend the order.

```
    public CarBoxData getNewOwnerCarBoxData() {
        PublicKey25519Proposition proposition;
        if(proof.isSeller()) {
            proposition = new PublicKey25519Proposition(carSellOrderBoxToOpen.proposition().getOwnerPublicKeyBytes());
        } else {
            proposition = new PublicKey25519Proposition(carSellOrderBoxToOpen.proposition().getBuyerPublicKeyBytes());
        }
        return new CarBoxData(
                proposition,
                carSellOrderBoxToOpen.getVin(),
                carSellOrderBoxToOpen.getYear(),
                carSellOrderBoxToOpen.getModel(),
                carSellOrderBoxToOpen.getColor()
        );
    }
```

The second one, "getPaymentBoxData", creates a coin box with the payment of the order price to the seller (it will be used only if the buyer accepts the order):

```
    public RegularBoxData getPaymentBoxData() {
        return new RegularBoxData(
                new PublicKey25519Proposition(carSellOrderBoxToOpen.proposition().getOwnerPublicKeyBytes()),
                carSellOrderBoxToOpen.getPrice()
        );
    }
```

Also this time we have the methods to serialize and deserialize the object: since the CarBuyOrderInfo is a property of our transaction and the transaction can be serialied, we need to be able to serialize and deserialize it as well.

Now that we have seen how a transaction is built, you may wonder how it can be created and submitted to the sidechain. This could be achieved in several ways, depending on the needs of our application, e.g. by using an RPC command, a code defined trigger, an offline wallet that creates the byte-array of the transaction and sends it through the default API method '*transaction/sendTransaction*', ... 
One of the most common ways to support the creation of a custom transaction is by extending the default API endpoints, and add a new custom local wallet endpoint to let the user create it via HTTP: we will look at how to do that at the end of this chapter.


## Custom proof and proposition

A proposition is a locker of a box, and a proof is the unlocker.
If standard ones are not enough for your needs, with the SDK you can define custom propositions and proofs.

Inside our application, for example, we introduced the SellOrderProposition: it is composed by two public keys, and the corresponding proof (SellOrderSpendingProof) will be able to unlock it by providing only one of the two keys.

First of all let's see the SellOrderProposition:

```
@JsonView(Views.Default.class)
public final class SellOrderProposition implements ProofOfKnowledgeProposition<PrivateKey25519> {
    private static final int KEY_LENGTH = Ed25519.keyLength();

    // Specify json attribute name for the ownerPublicKeyBytes field.
    @JsonProperty("ownerPublicKey")
    private final byte[] ownerPublicKeyBytes;

    // Specify json attribute name for the buyerPublicKeyBytes field.
    @JsonProperty("buyerPublicKey")
    private final byte[] buyerPublicKeyBytes;

    public SellOrderProposition(byte[] ownerPublicKeyBytes, byte[] buyerPublicKeyBytes) {
        if(ownerPublicKeyBytes.length != KEY_LENGTH)
            throw new IllegalArgumentException(String.format("Incorrect ownerPublicKeyBytes length, %d expected, %d found", KEY_LENGTH, ownerPublicKeyBytes.length));

        if(buyerPublicKeyBytes.length != KEY_LENGTH)
            throw new IllegalArgumentException(String.format("Incorrect buyerPublicKeyBytes length, %d expected, %d found", KEY_LENGTH, buyerPublicKeyBytes.length));

        this.ownerPublicKeyBytes = Arrays.copyOf(ownerPublicKeyBytes, KEY_LENGTH);

        this.buyerPublicKeyBytes = Arrays.copyOf(buyerPublicKeyBytes, KEY_LENGTH);
    }


    @Override
    public byte[] pubKeyBytes() {
        return Arrays.copyOf(ownerPublicKeyBytes, KEY_LENGTH);
    }

    public byte[] getOwnerPublicKeyBytes() {
        return pubKeyBytes();
    }

    public byte[] getBuyerPublicKeyBytes() {
        return Arrays.copyOf(buyerPublicKeyBytes, KEY_LENGTH);
    }

    @Override
    public byte[] bytes() {
        return Bytes.concat(
                ownerPublicKeyBytes,
                buyerPublicKeyBytes
        );
    }

    public static SellOrderProposition parseBytes(byte[] bytes) {
        int offset = 0;

        byte[] ownerPublicKeyBytes = Arrays.copyOfRange(bytes, offset, offset + KEY_LENGTH);
        offset += KEY_LENGTH;

        byte[] buyerPublicKeyBytes = Arrays.copyOfRange(bytes, offset, offset + KEY_LENGTH);

        return new SellOrderProposition(ownerPublicKeyBytes, buyerPublicKeyBytes);

    }

    @Override
    public PropositionSerializer serializer() {
        return SellOrderPropositionSerializer.getSerializer();
    }

    @Override
    public int hashCode() {
        int result = Arrays.hashCode(ownerPublicKeyBytes);
        result = 31 * result + Arrays.hashCode(buyerPublicKeyBytes);
        return result;
    }

    @Override
    public boolean equals(Object obj) {
        if (obj == null)
            return false;
        if (!(obj instanceof SellOrderProposition))
            return false;
        if (obj == this)
            return true;
        SellOrderProposition that = (SellOrderProposition) obj;
        return Arrays.equals(ownerPublicKeyBytes, that.ownerPublicKeyBytes)
                && Arrays.equals(buyerPublicKeyBytes, that.buyerPublicKeyBytes);
    }
}
```

As you can see, a custom proposition can have a number of private fields (in our case the properties ownerPublicKeyBytes and  buyerPublicKeyBytes, which have also their getters methods getOwnerPublicKeyBytes() and getBuyerPublicKeyBytes()). 
In any case it must also:
- implement the ProofOfKnowledgeProposition interface, and define its method pubKeyBytes, that returns a byte representation of the public key of this proposition:

```
    @Override
    public byte[] pubKeyBytes() {
        return Arrays.copyOf(ownerPublicKeyBytes, KEY_LENGTH);
    }

 ```   

- provide the usual methods for serialization and deserialization: 
   -  byte[] bytes()
   - parseBytes(byte[] bytes)
   -  serializer()

- implement correclty the methods hashCode() and equals(), used to compare this proposition with other ones.

 ```
    @Override
    public int hashCode() {
        int result = Arrays.hashCode(ownerPublicKeyBytes);
        result = 31 * result + Arrays.hashCode(buyerPublicKeyBytes);
        return result;
    }

    @Override
    public boolean equals(Object obj) {
        if (obj == null)
            return false;
        if (!(obj instanceof SellOrderProposition))
            return false;
        if (obj == this)
            return true;
        SellOrderProposition that = (SellOrderProposition) obj;
        return Arrays.equals(ownerPublicKeyBytes, that.ownerPublicKeyBytes)
                && Arrays.equals(buyerPublicKeyBytes, that.buyerPublicKeyBytes);
    }
 ```   

The corresponding proof class is the following:

 ``` 
public final class SellOrderSpendingProof extends AbstractSignature25519<PrivateKey25519, SellOrderProposition> {

    private final boolean isSeller;

    public static final int SIGNATURE_LENGTH = Ed25519.signatureLength();

    public SellOrderSpendingProof(byte[] signatureBytes, boolean isSeller) {
        super(signatureBytes);
        if (signatureBytes.length != SIGNATURE_LENGTH)
            throw new IllegalArgumentException(String.format("Incorrect signature length, %d expected, %d found", SIGNATURE_LENGTH,
                    signatureBytes.length));
        this.isSeller = isSeller;
    }

    public boolean isSeller() {
        return isSeller;
    }

    @Override
    public boolean isValid(SellOrderProposition proposition, byte[] message) {
        if(isSeller) {
            // Car seller wants to discard selling.
            return Ed25519.verify(signatureBytes, message, proposition.getOwnerPublicKeyBytes());
        } else {
            // Specific buyer wants to buy the car.
            return Ed25519.verify(signatureBytes, message, proposition.getBuyerPublicKeyBytes());
        }
    }

    @Override
    public byte proofTypeId() {
        return CarRegistryProofsIdsEnum.SellOrderSpendingProofId.id();
    }

    @Override
    public byte[] bytes() {
        return Bytes.concat(
                new byte[] { (isSeller ? (byte)1 : (byte)0) },
                signatureBytes
        );
    }

    public static SellOrderSpendingProof parseBytes(byte[] bytes) {
        int offset = 0;

        boolean isSeller = bytes[offset] != 0;
        offset += 1;

        byte[] signatureBytes = Arrays.copyOfRange(bytes, offset, offset + SIGNATURE_LENGTH);

        return new SellOrderSpendingProof(signatureBytes, isSeller);
    }

    @Override
    public ProofSerializer serializer() {
        return SellOrderSpendingProofSerializer.getSerializer();
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        SellOrderSpendingProof that = (SellOrderSpendingProof) o;
        return Arrays.equals(signatureBytes, that.signatureBytes) && isSeller == that.isSeller;
    }

    @Override
    public int hashCode() {
        int result = Objects.hash(signatureBytes.length);
        result = 31 * result + Arrays.hashCode(signatureBytes);
        result = 31 * result + (isSeller ? 1 : 0);
        return result;
    }
}
 ``` 

 The most important method here is  **isValid**: it receives a proposition and a byte[] message and checks that the signature contained in this proof (was passed in the constructor) is valid against them. If this method returns true, any box locked with the proposition can be opened with this proof.

 ``` 
     @Override
    public boolean isValid(SellOrderProposition proposition, byte[] message) {
        if(isSeller) {
            // Car seller wants to discard selling.
            return Ed25519.verify(
                signatureBytes, message, proposition.getOwnerPublicKeyBytes()
            );
        } else {
            // Specific buyer wants to buy the car.
            return Ed25519.verify(
                signatureBytes, message, proposition.getBuyerPublicKeyBytes()
            );
        }
    }    
 ``` 

 The other methdos are the usual ones: *proofTypeId* that returs a unique identifier of this proof type

 ```
    @Override
    public byte proofTypeId() {
        return CarRegistryProofsIdsEnum.SellOrderSpendingProofId.id();
    }
 ```  
the methods to compare the proof with other ones:

```
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        SellOrderSpendingProof that = (SellOrderSpendingProof) o;
        return Arrays.equals(signatureBytes, that.signatureBytes) && isSeller == that.isSeller;
    }

    @Override
    public int hashCode() {
        int result = Objects.hash(signatureBytes.length);
        result = 31 * result + Arrays.hashCode(signatureBytes);
        result = 31 * result + (isSeller ? 1 : 0);
        return result;
    }
 ```   


 and the methods to serialize and deserialize it:

 ```
    @Override
    public byte[] bytes() {
        return Bytes.concat(
                new byte[] { (isSeller ? (byte)1 : (byte)0) },
                signatureBytes
        );
    }

    public static SellOrderSpendingProof parseBytes(byte[] bytes) {
        int offset = 0;

        boolean isSeller = bytes[offset] != 0;
        offset += 1;

        byte[] signatureBytes = Arrays.copyOfRange(bytes, offset, offset + SIGNATURE_LENGTH);

        return new SellOrderSpendingProof(signatureBytes, isSeller);
    }

    @Override
    public ProofSerializer serializer() {
        return SellOrderSpendingProofSerializer.getSerializer();
    }
 ```  

Now that we have looked at all the different objects, we can understand better how the relationship between proposition, proofs and boxes is always expressed through the generics used when declaring them. For example the SellOrderProposition (first row below) appears also in the declaration of the related proof  and in the declaration of the CarSellOrderBox that gets locked by it:

public final class **SellOrderProposition** implements ProofOfKnowledgeProposition<PrivateKey25519> 

public final class SellOrderSpendingProof extends AbstractSignature25519<PrivateKey25519, **SellOrderProposition**> 

public final class CarSellOrderBox extends AbstractNoncedBox<**SellOrderProposition**, CarSellOrderBoxData, CarSellOrderBox> 

In this way, if you miss something in the design you can already get some errors at compile time.

























Extend API: 
***********

* Create a new class CarAPI which extends ApplicationAPIGroup class. Add this new class to route it in SimpleAppModule, as described in the Custom API manual. In our case it is done in ``CarRegistryAppModule`` by 

    * Creating ``customApiGroups`` as a list of custom API Groups:
    * ``List<ApplicationApiGroup> customApiGroups = new ArrayList<>()````;

    * Adding created ``CarApi`` into ``customApiGroups: customApiGroups.add(new CarApi())``;

    * Binding that custom api group via dependency injection:
      ::
       bind(new TypeLiteral<List<ApplicationApiGroup>> () {})
               .annotatedWith(Names.named("CustomApiGroups"))
               .toInstance(customApiGroups);


* Define Car creation transaction.

    * Defining request class/JSON request body
      As input for the transaction we expected: 
      Regular box id  as input for paying fee; 
      Fee value; 
      Proposition address which will be recognized as a Car Proposition; 
      Vehicle identification number of car. So next request class shall be created:
      :: 
       public class CreateCarBoxRequest {
       public String vin;
       public int year;
       public String model;
       public String color;
       public String proposition; // hex representation of public key proposition
       public long fee;

       // Setters to let Akka Jackson JSON library to automatically deserialize the request body.
            public void setVin(String vin) {
                this.vin = vin;
            }

            public void setYear(int year) {
                this.year = year;
            }

            public void setModel(String model) {
                this.model = model;
            }

            public void setColor(String color) {
                this.color = color;
            }

            public void setProposition(String proposition) {
                this.proposition = proposition;
            }

            public void setFee(long fee) {
                this.fee = fee;
            }
        }


Request class shall have appropriate setters and getters for all class members. Class members' names define a structure for related JSON structure according to `Jackson library <https://github.com/FasterXML/jackson-databind/>`_, so next JSON structure is expected to be set: 
::
 {
    "vin":"30124",
    “year”:1984,
    “model”: “Lamborghini”
    “color”:”deep black”
    "carProposition":"a5b10622d70f094b7276e04608d97c7c699c8700164f78e16fe5e8082f4bb2ac",
    "fee": 1,
    "boxId": "d59f80b39d24716b4c9a54cfed4bff8e6f76597a7b11761d0d8b7b27ddf8bd3c"
 }
        
A few notes: setter’s input parameter could have a different type than set class member. It allows us to make all necessary conversion in setters.


Define the response for the car creation transaction, the result of transaction shall be defined by implementing the SuccessResponse interface with the class members. Class members will be returned as an API response. All members will have properly set getters and the response class will have proper annotation ``@JsonView(Views.Default.class)`` thus the Jackson library is able to correctly represent the response class in JSON format. In our case, we expect to return transaction bytes. The response class is next:
::
    @JsonView(Views.Default.class)
    class TxResponse implements SuccessResponse {
    public String transactionBytes;
        public TxResponse(String transactionBytes) {
            this.transactionBytes = transactionBytes;
        }
    }

* Define Car creation transaction itself
  ::
   private ApiResponse createCar(SidechainNodeView view, CreateCarBoxRequest ent)

As a first parameter we pass reference to SidechainNodeView, second reference is previously defined class on step 1 for representation of JSON request. 

* Define the request for the CarSellOrder transaction with a CreateCarSellOrderRequest as we did for the car creation transaction request.

    * Define request class for Car sell order transaction CreateCarSellOrderRequest as it was done for Car creation transaction request:
      ::
       public class CreateCarSellOrderRequest {
        public String carBoxId; // hex representation of box id
        public String buyerProposition; // hex representation of public key proposition
        public long sellPrice;
        public long fee;

        // Setters to let Akka Jackson JSON library to automatically deserialize the request body.

        public void setCarBoxId(String carBoxId) {
            this.carBoxId = carBoxId;
        }

        public void setBuyerProposition(String buyerProposition) {
            this.buyerProposition = buyerProposition;
        }

        public void setSellPrice(long sellPrice) {
            this.sellPrice = sellPrice;
        }

        public void setFee(int fee) {
            this.fee = fee;
        }
       }

* Define Car Sell order transaction itself -- ``private ApiResponse createCarSellOrder(SidechainNodeView view, CreateCarSellOrderRequest ent)`` Required actions are similar as it was done to Create Car transaction. The main idea is a moving Car Box into CarSellOrderBox.

* Define Car sell order response --  As a result of Car sell order we could still use TxResponse
 
* Create AcceptCarSellorder transaction
    * Specify request as  
      ::
       public class SpendCarSellOrderRequest {
        public String carSellOrderId; // hex representation of box id
        public long fee;
        // Setters to let the Akka Jackson JSON library automatically deserialize the request body.
        public void setCarSellOrderId(String carSellOrderId) {
        this.carSellOrderId = carSellOrderId;
        }

        public void setFee(long fee) {
        this.fee = fee;
        }
       }
            
    * Specify acceptCarSellOrder transaction itself
    * As a result we still could use TxResponse class
    * Important part is creation proof for BuyCarTransaction, because we accept car buying then we shall form proof with defining that we buy car:
        ::
            
            SellOrderSpendingProof buyerProof = new SellOrderSpendingProof(
            buyerSecretOption.get().sign(messageToSign).bytes(),
            isSeller
            );
            
    Where *isSeller* is false.

* Create cancelCarSellOrder transaction
    * Specify cancel request as 
      ::
        public class SpendCarSellOrderRequest {
            public String carSellOrderId; // hex representation of box id
            public long fee;

            // Setters to let Akka Jackson JSON library to automatically deserialize the request body.

            public void setCarSellOrderId(String carSellOrderId) {
                this.carSellOrderId = carSellOrderId;
            }

            public void setFee(long fee) {
                this.fee = fee;
            }
        }
    * Specify the transaction itself. Because we recalled our sell order, the isSeller parameter during transaction creation is set to false.




