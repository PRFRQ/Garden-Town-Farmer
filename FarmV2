pragma solidity >=0.6.0 <0.8.0;
pragma experimental ABIEncoderV2;

//import "@openzeppelin/contracts/math/Math.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v3.3.0/contracts/math/Math.sol";

import "./Token.sol";

// Items, NFTs or resources
interface ERCItem {
    function mint(address account, uint256 amount) external;
    function burn(address account, uint256 amount) external;
    function balanceOf(address acount) external returns (uint256);
    
    // Used by resources - items/NFTs won't have this will this be an issue?
    function stake(address account, uint256 amount) external;
    function getStaked(address account) external returns(uint256);
}

contract FarmV2 {
    using SafeMath for uint256;

    TokenV2 private token;

    struct Square {
        Fruit fruit;
        uint createdAt;
    }

    struct V1Farm {
        address account;
        uint tokenAmount;
        uint size;
        Fruit fruit;
    }

    address burn =0x000000000000000000000000000000000000dEaD;

    uint farmCount = 0;
    bool isMigrating = true;
    mapping(address => Square[]) fields;
    mapping(address => uint) syncedAt;
    mapping(address => uint) rewardsOpenedAt;

    mapping(address => uint256) lastWatering;
    uint256 wateringTime  = 60 * 60 * 24 * 3; // 3 days
    uint256 wateringPrice = 35;
    address wateringReceiver = 0x000000000000000000000000000000000000dEaD;
    
    constructor(TokenV2 _token) public {
        token = _token;
    }
    
    // Need to upload these in batches so separate from constructor
    function uploadV1Farms(V1Farm[] memory farms) public {
        require(isMigrating, "MIGRATION_COMPLETE");

        uint decimals = token.decimals();
        
        // Carry over farms from V1
        for (uint i=0; i < farms.length; i += 1) {
            V1Farm memory farm = farms[i];

            Square[] storage land = fields[farm.account];
            
            // Treat them with a ripe plant
            Square memory plant = Square({
                fruit: farm.fruit,
                createdAt: 0
            });
            
            for (uint j=0; j < farm.size; j += 1) {
                land.push(plant);
            }

            syncedAt[farm.account] = block.timestamp;
            rewardsOpenedAt[farm.account] = block.timestamp;
            
            token.mint(farm.account, farm.tokenAmount * (10**decimals));
            
            farmCount += 1;
        }
    }
    
    function finishMigration() public {
        isMigrating = false;
    }
    
    event FarmCreated(address indexed _address);
    event FarmSynced(address indexed _address);
    event ItemCrafted(address indexed _address, address _item);

    // Function to receive Ether. msg.data must be empty
    receive() external payable {}

    function createFarm(address payable _charity) public payable {
        require(syncedAt[msg.sender] == 0, "FARM_EXISTS");

        uint decimals = token.decimals();

        require(
            // Donation must be at least $0.10 to play
            msg.value >= 1 * 10**(decimals - 1),
            "INSUFFICIENT_DONATION"
        );

        require(
            // The Water Project - double check
            _charity == address(0xD61A9BC2A46df1e24c6C224f4F89aB93a39dE631),"INVALID_CHARITY");


        Square[] storage land = fields[msg.sender];
        Square memory empty = Square({
            fruit: Fruit.None,
            createdAt: 0
        });
        Square memory sunflower = Square({
            fruit: Fruit.Sunflower,
            createdAt: 0
        });

        // Each farmer starts with 5 fields & 3 Sunflowers
        land.push(empty);
        land.push(sunflower);
        land.push(sunflower);
        land.push(sunflower);
        land.push(empty);

        syncedAt[msg.sender] = block.timestamp;
        // They must wait X days before opening their first reward
        rewardsOpenedAt[msg.sender] = block.timestamp;

        (bool sent, bytes memory data) = _charity.call{value: msg.value}("");
        require(sent, "DONATION_FAILED");

        farmCount += 1;
            
        //Emit an event
        emit FarmCreated(msg.sender);

        if(lastWatering[msg.sender]==0){
            lastWatering[msg.sender]=block.timestamp;
        }
    }
    
    function lastSyncedAt(address owner) private view returns(uint) {
        return syncedAt[owner];
    }


    function getLand(address owner) public view returns (Square[] memory) {
        return fields[owner];
    }

    enum Action { Plant, Harvest }
    enum Fruit { None, Sunflower, Potato, Pumpkin, Beetroot, Cauliflower, Parsnip, Radish }

    struct Event { 
        Action action;
        Fruit fruit;
        uint landIndex;
        uint createdAt;
    }

    struct Farm {
        Square[] land;
        uint balance;
    }

    function getHarvestSeconds(Fruit _fruit) private pure returns (uint) {
        if (_fruit == Fruit.Sunflower) {
            // 1 minute
            return 1 * 60;
        } else if (_fruit == Fruit.Potato) {
            // 5 minutes
            return 5 * 60;
        } else if (_fruit == Fruit.Pumpkin) {
            // 1 hour
            return 1  * 60 * 60;
        } else if (_fruit == Fruit.Beetroot) {
            // 4 hours
            return 4 * 60 * 60;
        } else if (_fruit == Fruit.Cauliflower) {
            // 8 hours
            return 8 * 60 * 60;
        } else if (_fruit == Fruit.Parsnip) {
            // 1 day
            return 24 * 60 * 60;
        } else if (_fruit == Fruit.Radish) {
            // 3 days
            return 3 * 24 * 60 * 60;
        }

        require(false, "INVALID_FRUIT");
        return 9999999;
    }

    function getSeedPrice(Fruit _fruit) private view returns (uint price) {
        uint decimals = token.decimals();

        if (_fruit == Fruit.Sunflower) {
            //$0.016
            return 16 * 10**decimals / 1000;
        } else if (_fruit == Fruit.Potato) {
            // $0.07
            return 7 * 10**decimals / 100;
        } else if (_fruit == Fruit.Pumpkin) {
            // $0.6
            return 60 * 10**decimals / 100;
        } else if (_fruit == Fruit.Beetroot) {
            // $2.3
            return 23 * 10**decimals / 10;
        } else if (_fruit == Fruit.Cauliflower) {
            // $4.8
            return 48 * 10**decimals / 10;
        } else if (_fruit == Fruit.Parsnip) {
            // $15
            return 15 * 10**decimals;
        } else if (_fruit == Fruit.Radish) {
            // $40
            return 40 * 10**decimals;
        }

        require(false, "INVALID_FRUIT");

        return 100000 * 10**decimals;
    }

    function getFruitPrice(Fruit _fruit) private view returns (uint price) {
        uint decimals = token.decimals();

        if (_fruit == Fruit.Sunflower) {
            // $0.03
            return 3 * 10**decimals / 100;
        } else if (_fruit == Fruit.Potato) {
            // $0.14
            return 14 * 10**decimals / 100;
        } else if (_fruit == Fruit.Pumpkin) {
            // $1.2
            return 12 * 10**decimals / 10;
        } else if (_fruit == Fruit.Beetroot) {
            // $4.7
            return 470 * 10**decimals / 100;
        } else if (_fruit == Fruit.Cauliflower) {
            // $10
            return 10 * 10**decimals;
        } else if (_fruit == Fruit.Parsnip) {
            // $35
            return 35 * 10**decimals;
        } else if (_fruit == Fruit.Radish) {
            // $100
            return 100 * 10**decimals;
        }

        require(false, "INVALID_FRUIT");

        return 0;
    }
    
    function requiredLandSize(Fruit _fruit) private pure returns (uint size) {
        if (_fruit == Fruit.Sunflower || _fruit == Fruit.Potato) {
            return 5;
        } else if (_fruit == Fruit.Pumpkin || _fruit == Fruit.Beetroot) {
            return 8;
        } else if (_fruit == Fruit.Cauliflower) {
            return 11;
        } else if (_fruit == Fruit.Parsnip) {
            return 14;
        } else if (_fruit == Fruit.Radish) {
            return 17;
        }

        require(false, "INVALID_FRUIT");

        return 99;
    }
    
       
    function getLandPrice(uint landSize) private view returns (uint price) {
        uint decimals = token.decimals();
        if (landSize <= 5) {
            // $50
            return 50 * 10**decimals;
        } else if (landSize <= 8) {
            // 100
            return 100 * 10**decimals;
        } else if (landSize <= 11) {
            // $150
            return 150 * 10**decimals;
        }
        
        // $200
        return 200 * 10**decimals;
    }

    modifier hasFarm {
        require(lastSyncedAt(msg.sender) > 0, "NO_FARM");
        _;
    }
     
    uint private THIRTY_MINUTES = 30 * 60;

    function buildFarm(Event[] memory _events) private view hasFarm returns (Farm memory currentFarm) {
        Square[] memory land = fields[msg.sender];
        uint balance = token.balanceOf(msg.sender);
        
        for (uint index = 0; index < _events.length; index++) {
            Event memory farmEvent = _events[index];

            uint thirtyMinutesAgo = block.timestamp.sub(THIRTY_MINUTES); 
            require(farmEvent.createdAt >= thirtyMinutesAgo, "EVENT_EXPIRED");
            require(farmEvent.createdAt >= lastSyncedAt(msg.sender), "EVENT_IN_PAST");
            require(farmEvent.createdAt <= block.timestamp, "EVENT_IN_FUTURE");

            if (index > 0) {
                require(farmEvent.createdAt >= _events[index - 1].createdAt, "INVALID_ORDER");
            }

            if (farmEvent.action == Action.Plant) {
                require(land.length >= requiredLandSize(farmEvent.fruit), "INVALID_LEVEL");
                require(checkWateringTime()==false,"WATERING EXPIRED");

                uint price = getSeedPrice(farmEvent.fruit);
                uint fmcPrice = getMarketPrice(price);
                require(balance >= fmcPrice, "INSUFFICIENT_FUNDS");

                balance = balance.sub(fmcPrice);

                Square memory plantedSeed = Square({
                    fruit: farmEvent.fruit,
                    createdAt: farmEvent.createdAt
                });
                land[farmEvent.landIndex] = plantedSeed;
            } else if (farmEvent.action == Action.Harvest) {
                Square memory square = land[farmEvent.landIndex];
                require(square.fruit != Fruit.None, "NO_FRUIT");
                require(checkWateringTime()==false,"WATERING EXPIRED");

                uint duration = farmEvent.createdAt.sub(square.createdAt);
                uint secondsToHarvest = getHarvestSeconds(square.fruit);
                require(duration >= secondsToHarvest, "NOT_RIPE");

                // Clear the land
                Square memory emptyLand = Square({
                    fruit: Fruit.None,
                    createdAt: 0
                });
                land[farmEvent.landIndex] = emptyLand;

                uint price = getFruitPrice(square.fruit);
                uint fmcPrice = getMarketPrice(price);

                balance = balance.add(fmcPrice);
            }
        }

        return Farm({
            land: land,
            balance: balance
        });
    }


    function sync(Event[] memory _events) public hasFarm returns (Farm memory) {
        Farm memory farm = buildFarm(_events);

        // Update the land
        Square[] storage land = fields[msg.sender];
        for (uint i=0; i < farm.land.length; i += 1) {
            land[i] = farm.land[i];
        }
        
        syncedAt[msg.sender] = block.timestamp;
        
        uint balance = token.balanceOf(msg.sender);
        // Update the balance - mint or burn
        if (farm.balance > balance) {
            uint profit = farm.balance.sub(balance);
            token.mint(msg.sender, profit);
        } else if (farm.balance < balance) {
            uint loss = balance.sub(farm.balance);
            token.burn(msg.sender, loss);
        }

        emit FarmSynced(msg.sender);

        return farm;
    }

    function levelUp() public hasFarm {
        require(fields[msg.sender].length <= 17, "MAX_LEVEL");

        Square[] storage land = fields[msg.sender];

        uint price = getLandPrice(land.length);
        uint fmcPrice = getMarketPrice(price);
        uint balance = token.balanceOf(msg.sender);

        require(balance >= fmcPrice, "INSUFFICIENT_FUNDS");
        
        token.transferFrom(msg.sender, burn, fmcPrice);
        
        // Add 3 sunflower fields in the new fields
        Square memory sunflower = Square({
            fruit: Fruit.Sunflower,
            // Make them immediately harvestable in case they spent all their tokens
            createdAt: 0
        });

        for (uint index = 0; index < 3; index++) {
            land.push(sunflower);
        }

        emit FarmSynced(msg.sender);
    }

    // watering system
    function watering() public hasFarm {
        uint decimals = token.decimals();

        token.transferFrom(msg.sender,wateringReceiver,wateringPrice*10**decimals);
        lastWatering[msg.sender]=block.timestamp;
    }

    function checkWateringTime()internal view returns(bool watstate){
        bool wateringState;
        if(block.timestamp>lastWatering[msg.sender].add(wateringTime)){
            wateringState=false; // need watering
        }
        else{
            wateringState=true; 
        }
        return wateringState;
    }

    function getWateringTime(address from)public view returns(uint256 time){
        return (lastWatering[from]+wateringTime)-block.timestamp;
    }

    function getWateringPrice()public view returns(uint256 price){
        uint decimals = token.decimals();
        return wateringPrice*10**decimals;
    }

    // How many tokens do you get per dollar
    // Algorithm is totalSupply / 10000 but we do this in gradual steps to avoid widly flucating prices between plant & harvest
    function getMarketRate() private view returns (uint conversion) {
        uint decimals = token.decimals();
        uint totalSupply = token.totalSupply();

        // Less than 100, 000 tokens
        // 1 Farm Dollar gets you 1 FMC token
        return 1;
        
    }

    function getMarketPrice(uint price) public view returns (uint conversion) {
        uint marketRate = getMarketRate();

        return price.div(marketRate);
    }
    
    function getFarm(address account) public view returns (Square[] memory farm) {
        return fields[account];
    }
    
    function getFarmCount() public view returns (uint count) {
        return farmCount;
    }

    /**
        Multi-token economy configurability below
     */
    // An in game material - Crafted Item, NFT or resource
    struct Material {
        address materialAddress;
        bool exists;
    }
    
    struct Cost {
        address materialAddress;
        uint amount;
    }

    struct Recipe {
        address outputAddress;
        Cost[] costs;
    }

    struct Resource {
        address outputAddress;
        address inputAddress;
    }

    mapping(address => Resource) resources;
    mapping(address => Recipe) recipes;
    mapping(address => Material) materials;

    // Put down a resource - tokens have their own mechanism for reflecting rewards
    function stake(address resourceAddress, uint amount) public {
        Material memory material = materials[resourceAddress];
        require(material.exists, "RESOURCE_DOES_NOT_EXIST");

        Resource memory resource = resources[resourceAddress];

        ERCItem(resource.inputAddress).burn(msg.sender, amount);


        // The resource contract will determine tokenomics and what to do with staked amount
        ERCItem(resource.outputAddress).stake(msg.sender, amount);
    }

    function createRecipe(address tokenAddress, Cost[] memory costs) public {
        require(tokenAddress != address(token), "SUNFLOWER_TOKEN_IN_USE");
        require(!materials[tokenAddress].exists, "RECIPE_ALREADY_EXISTS");

        // Ensure all materials are setup
        for (uint i=0; i < costs.length; i += 1) {
            address input = costs[i].materialAddress;
            Material memory material = materials[input];

            require(input == address(token) || material.exists, "MATERIAL_DOES_NOT_EXIST");
            
            recipes[tokenAddress].costs.push(costs[i]);
        }

        materials[tokenAddress] = Material({
            exists: true,
            materialAddress: tokenAddress
        });
    }

    function createResource(address resourceAddress, address requires) public {
        require(resourceAddress != address(token), "SUNFLOWER_TOKEN_IN_USE");
        require(!materials[resourceAddress].exists, "RESOURCE_ALREADY_EXISTS");

        // Check the required material is setup
        require(materials[requires].exists, "MATERIAL_DOES_NOT_EXIST");

        resources[resourceAddress] = Resource({
            outputAddress: resourceAddress,
            inputAddress: requires
        });
        
        materials[resourceAddress] = Material({
            exists: true,
            materialAddress: resourceAddress
        });
    }

    function burnCosts(address recipeAddress, uint total) private {
        require(total<10000);
        Recipe memory recipe = recipes[recipeAddress];

        // ERC20 contracts will validate as needed
        for (uint i=0; i < recipe.costs.length; i += 1) {
            Cost memory cost = recipe.costs[i];

            uint price = cost.amount * total;

            // Never burn SFF - Store rewards in the Farm Contract to redistribute
            if (cost.materialAddress == address(token)) {
                token.transferFrom(msg.sender, address(this), price);
            } else {
                ERCItem(cost.materialAddress).burn(msg.sender, price);
            }
        }
    }

    function craft(address recipeAddress, uint amount) public {
        require(amount<10000);
        Material memory material = materials[recipeAddress];
                
        require(material.exists, "RECIPE_DOES_NOT_EXIST");
        
        burnCosts(recipeAddress, amount);

        ERCItem(recipeAddress).mint(msg.sender, amount);
        
        emit ItemCrafted(msg.sender, recipeAddress);
    }

    function mintNFT(address recipeAddress, uint tokenId) public {
        Material memory material = materials[recipeAddress];
                
        require(material.exists, "RECIPE_DOES_NOT_EXIST");
        
        burnCosts(recipeAddress, 1);

        ERCItem(recipeAddress).mint(msg.sender, tokenId);
        
        emit ItemCrafted(msg.sender, recipeAddress);
    }
    
    function getRecipe(address recipeAddress) public view returns (Recipe memory recipe) {
        return recipes[recipeAddress];
    }

    function getResource(address resourceAddress) public view returns (Resource memory resource) {
        return resources[resourceAddress];
    }
}
