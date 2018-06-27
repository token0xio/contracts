# Smart Contracts

Example of Smart Contracts of Token0x


```Javascript
pragma solidity ^0.4.21;

contract Ownable {

    address public owner;

    function Ownable() public {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    function transferOwnership(address newOwner) public onlyOwner {
        owner = newOwner;
    }
}

contract MultiOwnable {

    mapping (address => bool) owners;

    modifier onlyOwner() {
        require(isOwner(msg.sender));
        _;
    }

    function MultiOwnable() public {
        address owner = msg.sender;
        owners[owner] = true;
    }

    function isOwner(address owner) public view returns (bool) {
        return owners[owner];
    }

    function addOwner(address owner) onlyOwner public returns (bool) {
        require(!isOwner(owner) && owner != address(0));
        owners[owner] = true;
        return true;
    }

    function removeOwner(address owner) onlyOwner public returns (bool) {
        require(isOwner(owner));
        owners[owner] = false;
        return true;
    }
}

contract SafeMath {

    function safeMul(uint256 a, uint256 b) internal pure returns (uint256 ) {
        uint256 c = a * b;
        assert(a == 0 || c / a == b);
        return c;
    }

    function safeDiv(uint256 a, uint256 b) internal pure returns (uint256 ) {
        assert(b > 0);
        uint256 c = a / b;
        assert(a == b * c + a % b);
        return c;
    }

    function safeSub(uint256 a, uint256 b) internal pure returns (uint256 ) {
        assert(b <= a);
        return a - b;
    }

    function safeAdd(uint256 a, uint256 b) internal pure returns (uint256 ) {
        uint256 c = a + b;
        assert(c >= a);
        return c;
    }
}

contract ERC20 {

    /* This is a slight change to the ERC20 base standard.
    function totalSupply() constant returns (uint256 supply);
    is replaced with:
    uint256 public totalSupply;
    This automatically creates a getter function for the totalSupply.
    This is moved to the base contract since public getter functions are not
    currently recognised as an implementation of the matching abstract
    function by the compiler.
    */
    /// total amount of tokens
    uint256 public totalSupply;

    /// @param _owner The address from which the balance will be retrieved
    /// @return The balance
    function balanceOf(address _owner) public constant returns (uint256 balance);

    /// @notice send `_value` token to `_to` from `msg.sender`
    /// @param _to The address of the recipient
    /// @param _value The amount of token to be transferred
    /// @return Whether the transfer was successful or not
    function transfer(address _to, uint256 _value) public returns (bool success);

    /// @notice send `_value` token to `_to` from `_from` on the condition it is approved by `_from`
    /// @param _from The address of the sender
    /// @param _to The address of the recipient
    /// @param _value The amount of token to be transferred
    /// @return Whether the transfer was successful or not
    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success);

    /// @notice `msg.sender` approves `_spender` to spend `_value` tokens
    /// @param _spender The address of the account able to transfer the tokens
    /// @param _value The amount of tokens to be approved for transfer
    /// @return Whether the approval was successful or not
    function approve(address _spender, uint256 _value) public returns (bool success);

    /// @param _owner The address of the account owning tokens
    /// @param _spender The address of the account able to transfer the tokens
    /// @return Amount of remaining tokens allowed to spent
    function allowance(address _owner, address _spender) public constant returns (uint256 remaining);

    event Transfer(address indexed _from, address indexed _to, uint256 _value);
    event Approval(address indexed _owner, address indexed _spender, uint256 _value);
}

contract StandardToken is ERC20, SafeMath {

    mapping(address => uint256) balances;
    mapping(address => mapping(address => uint256)) allowed;

    /// @dev Returns number of tokens owned by given address.
    /// @param _owner Address of token owner.
    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }

    /// @dev Transfers sender's tokens to a given address. Returns success.
    /// @param _to Address of token receiver.
    /// @param _value Number of tokens to transfer.
    function transfer(address _to, uint256 _value) public returns (bool) {
        require(_to != address(0));
        require(_value <= balances[msg.sender]);

        balances[msg.sender] = safeSub(balances[msg.sender], _value);
        balances[_to] = safeAdd(balances[_to], _value);
        emit Transfer(msg.sender, _to, _value);
        return true;
    }

    /// @dev Allows allowed third party to transfer tokens from one address to another. Returns success.
    /// @param _from Address from where tokens are withdrawn.
    /// @param _to Address to where tokens are sent.
    /// @param _value Number of tokens to transfer.
    function transferFrom(address _from, address _to, uint256 _value) public returns (bool) {
        require(_to != address(0));
        require(_value <= balances[_from]);
        require(_value <= allowed[_from][msg.sender]);
        
        balances[_to] = safeAdd(balances[_to], _value);
        balances[_from] = safeSub(balances[_from], _value);
        allowed[_from][msg.sender] = safeSub(allowed[_from][msg.sender], _value);
        emit Transfer(_from, _to, _value);
        return true;
    }

    /// @dev Sets approved amount of tokens for spender. Returns success.
    /// @param _spender Address of allowed account.
    /// @param _value Number of approved tokens.
    function approve(address _spender, uint256 _value) public returns (bool) {
        allowed[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    /// @dev Returns number of allowed tokens for given address.
    /// @param _owner Address of token owner.
    /// @param _spender Address of token spender.
    function allowance(address _owner, address _spender) public constant returns (uint256 remaining) {
        return allowed[_owner][_spender];
    }
}

contract Token is StandardToken, MultiOwnable {

    string public name;
    string public symbol;
    uint256 public totalSupply;
    uint8 public decimals = 18;
    string public version = 'v0.1';

    address public seller;

    uint256 public totalTokensSold;

    bool public locked;
    bool public enabledDex;

    modifier onlyUnlocked() {
        require(isOwner(msg.sender) || !locked);
        _;
    }

    function Token(
        string _name,
        string _symbol,
        uint256 _totalSupply,
        bool _enabledDex
    ) public MultiOwnable() {

        locked = true;
        name = _name;
        symbol = _symbol;
        totalSupply = _totalSupply;
        totalTokensSold = 0;
        enabledDex = _enabledDex;
    }

    function initSeller(address _newSeller) public {
        
        require(seller == address(0));
        
        seller = _newSeller;
        balances[seller] = totalSupply;
        emit Transfer(address(0), seller, totalSupply);

    }
    
    function changeSeller(address newSeller) onlyOwner public {
        require(newSeller != address(0) && seller != newSeller);

        uint256 unsoldTokens = balances[seller];
        balances[seller] = 0;
        balances[newSeller] = safeAdd(balances[newSeller], unsoldTokens);
        emit Transfer(seller, newSeller, unsoldTokens);

        seller = newSeller;
    }

    function sell(address _to, uint256 _value) onlyOwner public {

        require(_to != address(0));
        require(_value > 0 && _value <= balances[seller]);

        balances[seller] = safeSub(balances[seller], _value);
        balances[_to] = safeAdd(balances[_to], _value);
        emit Transfer(seller, _to, _value);

        totalTokensSold = safeAdd(totalTokensSold, _value);
    }

    function transfer(address _to, uint256 _value) onlyUnlocked public returns (bool) {
        return super.transfer(_to, _value);
    }

    function transferFrom(address _from, address _to, uint256 _value) onlyUnlocked public returns (bool) {
        return super.transferFrom(_from, _to, _value);
    }
    
    

    function lock() onlyOwner public {
        locked = true;
    }

    function unlock() onlyOwner public {
        locked = false;
    }

    function burn(uint256 _value) public {
        
        require(_value > 0 && _value <= balances[msg.sender]);

        balances[msg.sender] = safeSub(balances[msg.sender], _value) ;
        totalSupply = safeSub(totalSupply, _value);
        emit Transfer(msg.sender, address(0), _value);
    }
    
    //exchange
    
    struct Offer {
        uint256 tokens;
        uint256 price;
    }
    
    mapping(address => Offer) exchange;
    
    uint256 public market = 0;
    address[] public salers; 
    
    function getOffer(address _owner) public view returns (uint256[2]) {
        Offer storage offer = exchange[_owner];
        return ([offer.price , offer.tokens]);
    }
    
    function setOffer(address _owner, uint256 _price, uint256 _value) internal {
        exchange[_owner].price = _price;
        market =  safeSub(market, exchange[_owner].tokens);
        exchange[_owner].tokens = _value;
        market =  safeAdd(market, _value);
        if (_value == 0) {
            removeByValue(_owner);
        }
        else {
            salers.push(_owner);
        }
    }
    
    function find(address value) internal view returns(uint) {
        uint i = 0;
        while (salers[i] != value) {
            i++;
        }
        return i;
    }

    function removeByValue(address value) internal {
        uint i = find(value);
        if (salers[i] == value) {
            removeByIndex(i);
        }
    }

    function removeByIndex(uint i) internal {
        while (i<salers.length-1) {
            salers[i] = salers[i+1];
            i++;
        }
        salers.length--;
    }
    
    function offerToSell(uint256 _price, uint256 _value) public {
        require(!locked);
        require(enabledDex);
        setOffer(msg.sender, _price, _value);
    }
    
    function executeOffer(address owner) public payable {
        require(!locked);
        require(enabledDex);
        Offer storage offer = exchange[owner];
        require(offer.tokens > 0);
        require(msg.value == offer.price);
        owner.transfer(msg.value);
        require(balances[owner] >= offer.tokens);
        balances[owner] = safeSub(balances[owner], offer.tokens);
        balances[msg.sender] = safeAdd(balances[msg.sender], offer.tokens);
        emit Transfer(owner, msg.sender, offer.tokens);
        setOffer(owner, 0, 0);
    }
    
}

contract Pausable is Ownable {

    bool public paused;

    modifier ifNotPaused {
        require(!paused);
        _;
    }

    modifier ifPaused {
        require(paused);
        _;
    }

    // Called by the owner on emergency, triggers paused state
    function pause() external onlyOwner {
        paused = true;
    }

    // Called by the owner on end of emergency, returns to normal state
    function resume() external onlyOwner ifPaused {
        paused = false;
    }
}

library StringUtils {
    /// @dev Does a byte-by-byte lexicographical comparison of two strings.
    /// @return a negative number if `_a` is smaller, zero if they are equal
    /// and a positive numbe if `_b` is smaller.
    function compare(string _a, string _b) returns (int) {
        bytes memory a = bytes(_a);
        bytes memory b = bytes(_b);
        uint minLength = a.length;
        if (b.length < minLength) minLength = b.length;
        //@todo unroll the loop into increments of 32 and do full 32 byte comparisons
        for (uint i = 0; i < minLength; i ++)
            if (a[i] < b[i])
                return -1;
            else if (a[i] > b[i])
                return 1;
        if (a.length < b.length)
            return -1;
        else if (a.length > b.length)
            return 1;
        else
            return 0;
    }
    /// @dev Compares two strings and returns true iff they are equal.
    function equal(string _a, string _b) returns (bool) {
        return compare(_a, _b) == 0;
    }
    /// @dev Finds the index of the first occurrence of _needle in _haystack
    function indexOf(string _haystack, string _needle) returns (int)
    {
    	bytes memory h = bytes(_haystack);
    	bytes memory n = bytes(_needle);
    	if(h.length < 1 || n.length < 1 || (n.length > h.length)) 
    		return -1;
    	else if(h.length > (2**128 -1)) // since we have to be able to return -1 (if the char isn't found or input error), this function must return an "int" type with a max length of (2^128 - 1)
    		return -1;									
    	else
    	{
    		uint subindex = 0;
    		for (uint i = 0; i < h.length; i ++)
    		{
    			if (h[i] == n[0]) // found the first char of b
    			{
    				subindex = 1;
    				while(subindex < n.length && (i + subindex) < h.length && h[i + subindex] == n[subindex]) // search until the chars don't match or until we reach the end of a or b
    				{
    					subindex++;
    				}	
    				if(subindex == n.length)
    					return int(i);
    			}
    		}
    		return -1;
    	}	
    }
}

contract Tokensale is SafeMath, Ownable, Pausable {
    using StringUtils for string;
    
    struct Contribution {
        address referral;
        uint256 amount;
        uint256 pendingTokens;
    }
    
    bool public isRequiredKYC;
    bool public whitelistEnabled;
    
    address public tokenAddress;    

    uint256 public minAmountWei;

    uint256 public totalTokensSold; 
    uint256 public totalReceivedWei;
    uint256 public totalReferralFeeWei;
    uint256 public referralFee;
    
    uint256 public minCapWei;
    uint256 public maxCapInTokens;  

    uint256 public startTime;       
    uint256 public endTime;         
    
    address public escrow;
    uint256 public escrowGrantedAmount;
    uint256 public escrowGrantedAmountTotal;
    uint256 public escrowFee;
    
    mapping (address => Contribution) contributions;

    function transferEscrow(address _escrow) public {
        
        require(_escrow != address(0));
        
        require(_escrow != escrow);
        
        require(escrow == msg.sender);
        
        escrow = _escrow;
        
    }

    function setToken(address _token) public onlyOwner {
        
        require(tokenAddress == address(0));
        
        tokenAddress = _token;
        
    }

    function Tokensale(uint256 _minAmountWei,
                         uint256 _maxCapInTokens,
                         uint256 _startTime,
                         uint256 _endTime,
                         address _escrow,
                         uint256 _escrowFee,
                         uint256 _minCapWei,
                         uint256 _referralFee,
                         bool _isRequiredKYC,
                         bool _whitelistEnabled) Ownable() public {
                         
        minAmountWei = _minAmountWei;
        minCapWei = _minCapWei;
        referralFee = _referralFee;
        maxCapInTokens = _maxCapInTokens;
        startTime = _startTime;
        endTime   = _endTime;
        escrow = _escrow;
        escrowFee = _escrowFee;
        isRequiredKYC = _isRequiredKYC;
        whitelistEnabled = _whitelistEnabled;
    }

    
    function bytesToAddress(bytes b) internal pure returns (address) {
        uint result = 0;
        for (uint i = 0; i < b.length; i++) {
            uint c = uint(b[i]);
            if (c >= 48 && c <= 57) {
                result = result * 16 + (c - 48);
            }
            if(c >= 65 && c<= 90) {
                result = result * 16 + (c - 55);
            }
            if(c >= 97 && c<= 122) {
                result = result * 16 + (c - 87);
            }
        }
        return address(result);
    }
    
    
    function whitelistBuy(
        string length,
        string token,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) payable public {
        require(Token(tokenAddress).symbol().equal(token));
        require(ecrecover(keccak256("\x19Ethereum Signed Message:\n", length, "buy", token, toString(msg.sender)), v, r, s) == escrow);
        buy();
    }
    
    function buy() internal {
    
        require(!paused);
        require(tokenAddress != address(0));
        require(startTime <= now && now <= endTime);
        require(msg.value >= minAmountWei);
        
        Contribution storage contrib = contributions[msg.sender];
        
        if (msg.data.length == 40 && contrib.referral == address(0) && contrib.amount == 0) {
            contrib.referral = bytesToAddress(msg.data);
        }
        
        
        if (contrib.referral != address(0)) {
            totalReferralFeeWei = safeAdd(totalReferralFeeWei, safeDiv(msg.value, referralFee));
        }
        
        contrib.amount = safeAdd(contrib.amount, msg.value);
        
        totalReceivedWei = safeAdd(totalReceivedWei, msg.value);
        
        uint256 tokens = exchangeRate(msg.value);
        
        totalTokensSold = safeAdd(totalTokensSold, tokens);
        
        require(totalTokensSold <= maxCapInTokens);
        
        if (isRequiredKYC) {
            contrib.pendingTokens = safeAdd(contrib.pendingTokens, tokens);
        }
        else {
            Token(tokenAddress).sell(msg.sender, tokens);
        }
    }
    
    function() payable {
        require(!whitelistEnabled);
        buy();
    }
    
    function char(byte b) internal view returns (byte c) {
       if (b < 10) return byte(uint8(b) + 0x30);
       else return byte(uint8(b) + 0x57);
    }
    
    function toString(address x) internal view returns (string) {
        bytes memory s = new bytes(40);
        for (uint i = 0; i < 20; i++) {
            byte b = byte(uint8(uint(x) / (2**(8*(19 - i)))));
            byte hi = byte(uint8(b) / 16);
            byte lo = byte(uint8(b) - 16 * uint8(hi));
            s[2*i] = char(hi);
            s[2*i+1] = char(lo);            
        }
        return string(s);
    }
    
    
    
    function claimTokens(
        string length,
        string token,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) {
        string memory address_str = toString(msg.sender);
        require(ecrecover(keccak256("\x19Ethereum Signed Message:\n", length, "cla", token, toString(msg.sender)), v, r, s) == escrow);
        
        Contribution storage contrib = contributions[msg.sender];
        require(contrib.pendingTokens > 0);
        require(Token(tokenAddress).symbol().equal(token));
        Token(tokenAddress).sell(msg.sender, contrib.pendingTokens);
        contrib.pendingTokens = 0;
    }
    
    function getReferalFee(address from) {
        
        require(totalReceivedWei >= minCapWei && endTime < now);
        
        Contribution storage contrib = contributions[from];
        
        require(contrib.referral == msg.sender);
        
        contrib.referral = address(0);
        
        contrib.referral.transfer(safeDiv(contrib.amount, referralFee));
        
    }
    
    function refund() {
        
        require(totalReceivedWei < minCapWei && endTime < now);
        
        Contribution storage contrib = contributions[msg.sender];
        
        require(contrib.amount > 0);
        
        contrib.amount = 0;
        
        totalReceivedWei = safeSub(totalReceivedWei, contrib.amount);
        
        address(msg.sender).transfer(contrib.amount);
    }

    function exchangeRate(uint256 _weiValue) public view returns (uint256);
    

    function escrowGrant(uint256 _amount) public {
        
        require(totalReceivedWei >= minCapWei);
        
        require(address(this).balance >= _amount);
        
        require(endTime < now || totalTokensSold == maxCapInTokens);
        
        require(msg.sender == escrow);
        
        uint256 total = safeSub(totalReceivedWei, totalReferralFeeWei);
        
        escrowGrantedAmount = _amount;
        escrowGrantedAmountTotal = safeAdd(escrowGrantedAmountTotal, _amount);
        require(total >= escrowGrantedAmountTotal);
    }

    function withdraw(address _beneficiary, uint256 _amount) public onlyOwner {
        
        require(escrowGrantedAmount >= _amount);
        
        escrowGrantedAmount = safeSub(escrowGrantedAmount, _amount);
        
        uint256 serviceFee = safeDiv(_amount, escrowFee);
        
        uint256 toBeneficiary = safeSub(_amount, serviceFee);
        
        escrow.transfer(serviceFee);
        
        _beneficiary.transfer(toBeneficiary);
    }
}

contract ProdToken is Token {

    function ProdToken() Token(
        "Ethnamed",
        "BCC",
        700000 ether,
        true
    ) public { }
}

contract ProdPrivateSale is SafeMath, Tokensale  {

    function ProdPrivateSale() Tokensale(
         1 ether,
         500000 ether,
         1529606460,
         1532284860,
         0xad69f2ffd7d7b3e82605f5fe80acc9152929f283,
         33,
         100 ether,
         20,
         true,
         true
    ) public { }
    
    function exchangeRate(uint256 _wei) public view returns (uint256) {
        
        if (_wei >= 150 ether )
            return safeMul(_wei, 500 );
        
        return safeMul(_wei, 500);
    }
}
```
