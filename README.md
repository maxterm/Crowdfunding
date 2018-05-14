contract token { mapping (address => uint) public coinBalanceOf; function token() {}  function sendCoin(address receiver, uint amount) returns(bool sufficient) {  } }

contract Crowdsale {

    address public beneficiary;
    uint public fundingGoal; uint public amountRaised; uint public deadline; uint public price;
    token public tokenReward;   
    Funder[] public funders;
    event FundTransfer(address backer, uint amount, bool isContribution);

    /* data structure to hold information about campaign contributors */
    struct Funder {
        address addr;
        uint amount;
    }

    /*  at initialization, setup the owner */
    function Crowdsale(address _beneficiary, uint _fundingGoal, uint _duration, uint _price, token _reward) {
        beneficiary = _beneficiary;
        fundingGoal = _fundingGoal;
        deadline = now + _duration * 1 minutes;
        price = _price;
        tokenReward = token(_reward);
    }   

    /* The function without name is the default function that is called whenever anyone sends funds to a contract */
    function () {
        uint amount = msg.value;
        funders[funders.length++] = Funder({addr: msg.sender, amount: amount});
        amountRaised += amount;
        tokenReward.sendCoin(msg.sender, amount / price);
        FundTransfer(msg.sender, amount, true);
    }

    modifier afterDeadline() { if (now >= deadline) _ }

    /* checks if the goal or time limit has been reached and ends the campaign */
    function checkGoalReached() afterDeadline {
        if (amountRaised >= fundingGoal){
            beneficiary.send(amountRaised);
            FundTransfer(beneficiary, amountRaised, false);
        } else {
            FundTransfer(0, 11, false);
            for (uint i = 0; i < funders.length; ++i) {
              funders[i].addr.send(funders[i].amount);  
              FundTransfer(funders[i].addr, funders[i].amount, false);
            }               
        }
        suicide(beneficiary);
    }
}

/* setting example */

var _beneficiary = eth.accounts[1];    // create an account for this
var _fundingGoal = web3.toWei(100, "ether"); // raises 100 ether
var _duration = 30;     // number of minutes the campaign will last
var _price = web3.toWei(0.02, "ether"); // the price of the tokens, in ether
var _reward = token.address;   // the token contract address.

/* Deploy  example */

var crowdsaleCompiled = eth.compile.solidity(' contract token { mapping (address => uint) public coinBalanceOf; function token() {} function sendCoin(address receiver, uint amount) returns(bool sufficient) { } } contract Crowdsale { address public beneficiary; uint public fundingGoal; uint public amountRaised; uint public deadline; uint public price; token public tokenReward; Funder[] public funders; event FundTransfer(address backer, uint amount, bool isContribution); /* data structure to hold information about campaign contributors */ struct Funder { address addr; uint amount; } /* at initialization, setup the owner */ function Crowdsale(address _beneficiary, uint _fundingGoal, uint _duration, uint _price, token _reward) { beneficiary = _beneficiary; fundingGoal = _fundingGoal; deadline = now + _duration * 1 minutes; price = _price; tokenReward = token(_reward); } /* The function without name is the default function that is called whenever anyone sends funds to a contract */ function () { Funder f = funders[++funders.length]; f.addr = msg.sender; f.amount = msg.value; amountRaised += f.amount; tokenReward.sendCoin(msg.sender, f.amount/price); FundTransfer(f.addr, f.amount, true); } modifier afterDeadline() { if (now >= deadline) _ } /* checks if the goal or time limit has been reached and ends the campaign */ function checkGoalReached() afterDeadline { if (amountRaised >= fundingGoal){ beneficiary.send(amountRaised); FundTransfer(beneficiary, amountRaised, false); } else { FundTransfer(0, 11, false); for (uint i = 0; i < funders.length; ++i) { funders[i].addr.send(funders[i].amount); FundTransfer(funders[i].addr, funders[i].amount, false); } } suicide(beneficiary); } }');

var crowdsaleContract = web3.eth.contract(crowdsaleCompiled.Crowdsale.info.abiDefinition);
var crowdsale = crowdsaleContract.new(
  _beneficiary, 
  _fundingGoal, 
  _duration, 
  _price, 
  _reward,
  {
    from:web3.eth.accounts[0], 
    data:crowdsaleCompiled.Crowdsale.code, 
    gas: 1000000
  }, function(e, contract){
    if(!e) {

      if(!contract.address) {
        console.log("Contract transaction send: TransactionHash: " + contract.transactionHash " waiting to be mined...");

      } else {
        console.log("Contract mined! Address: " + contract.address);
        console.log(contract);
      }

    }    })
	
/* watcher on */

var event = crowdsale.FundTransfer({}, '', function(error, result){
  if (!error)

    if (result.args.isContribution) {
        console.log("\n New backer! Received " + web3.fromWei(result.args.amount, "ether") + " ether from " + result.args.backer  )

        console.log( "\n The current funding at " +( 100 *  crowdsale.amountRaised.call() / crowdsale.fundingGoal.call()) + "% of its goals. Funders have contributed a total of " + web3.fromWei(crowdsale.amountRaised.call(), "ether") + " ether.");

        var timeleft = Math.floor(Date.now() / 1000)-crowdsale.deadline();
        if (timeleft>3600) {  console.log("Deadline has passed, " + Math.floor(timeleft/3600) + " hours ago")
        } else if (timeleft>0) {  console.log("Deadline has passed, " + Math.floor(timeleft/60) + " minutes ago")
        } else if (timeleft>-3600) {  console.log(Math.floor(-1*timeleft/60) + " minutes until deadline")
        } else {  console.log(Math.floor(-1*timeleft/3600) + " hours until deadline")
        }

    } else {
        console.log("Funds transferred from crowdsale account: " + web3.fromWei(result.args.amount, "ether") + " ether to " + result.args.backer  )
    }

});

/* Register Contract */
var name = "mycrowdsale"
registrar.addr(name) 
registrar.reserve.sendTransaction(name, {from: eth.accounts[0]});
registrar.setAddress.sendTransaction(name, crowdsale.address, true,{from: eth.accounts[0]});
	

/*Contribution two ways*/

var amount = web3.toWei(5, "ether") // decide how much to contribute

eth.sendTransaction({from: eth.accounts[0], to: crowdsale.address, value: amount, gas: 1000000})

eth.sendTransaction({from: eth.accounts[0], to: registrar.addr("mycrowdsale"), value: amount, gas: 500000})
