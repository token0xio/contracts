# contracts

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

    modifier onlyUnlocked() {
        require(isOwner(msg.sender) || !locked);
        _;
    }

    function Token(
        string _name,
        string _symbol,
        uint256 _totalSupply
    ) public MultiOwnable() {

        locked = true;
        name = _name;
        symbol = _symbol;
        totalSupply = _totalSupply;
        totalTokensSold = 0;
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

contract Tokensale is SafeMath, Ownable, Pausable {

    address public tokenAddress;    

    uint256 public minAmountWei;

    uint256 public totalTokensSold; 
    uint256 public maxCapInTokens;  

    uint256 public startTime;       
    uint256 public endTime;         
    
    address public escrow;
    uint256 public escrowGrantedAmount;
    uint256 public escrowFee;
    
    mapping (address => uint256) referrals;

    function transferEscrow(address _escrow) public {
        
        require(_escrow != address(0));
        
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
                         uint256 _escrowFee) Ownable() public {
                         
        minAmountWei = _minAmountWei;
        maxCapInTokens = _maxCapInTokens;
        startTime = _startTime;
        endTime   = _endTime;
        escrow = _escrow;
        escrowFee = _escrowFee;
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
    
    
    function() payable {
        
        require(!paused);
        require(tokenAddress != address(0));
        require(startTime <= now && now <= endTime);
        require(msg.value >= minAmountWei);
        
        if (msg.data.length == 40) {
            address refAddress = bytesToAddress(msg.data);
            referrals[refAddress] = safeAdd(referrals[refAddress], msg.value);
        }
        
        uint256 tokens = exchangeRate(msg.value);
        totalTokensSold = safeAdd(totalTokensSold, tokens);
        require(totalTokensSold <= maxCapInTokens);
        Token(tokenAddress).sell(msg.sender, tokens);
        
    }
    

    function exchangeRate(uint256 _weiValue) public view returns (uint256);

    function escrowGrant(uint256 _amount) public {
        
        require(msg.sender == escrow);
        
        escrowGrantedAmount = safeAdd(escrowGrantedAmount, _amount);
    }

    function withdraw(address _beneficiary, uint256 _amount) public onlyOwner {
        
        require(escrowGrantedAmount >= _amount);
        
        require(address(this).balance >= _amount);
        
        escrowGrantedAmount = safeSub(escrowGrantedAmount, _amount);
        
        uint256 serviceFee = safeDiv(_amount, escrowFee);
        
        uint256 toBeneficiary = safeSub(_amount, serviceFee);
        
        escrow.transfer(serviceFee);
        
        _beneficiary.transfer(toBeneficiary);
    }
}

contract ProdToken is Token {

    function ProdToken() Token(
        'Qurrex',
        'BTC',
        700000 ether
    ) public { }
}

contract ProdPrivateSale is SafeMath, Tokensale  {

    function ProdPrivateSale() Tokensale(
         1 ether,
         500000 ether,
         1523776560,
         1526454960,
         0xc836225dddc253b41e6cec6d0b3380e3e091a377,
         33
    ) public { }
    
    function exchangeRate(uint256 _wei) public view returns (uint256) {
        
        if (_wei >= 150 ether)
            return safeMul(_wei, 500);
        else if (_wei >= 30 ether)
            return safeMul(_wei, 500);
        else if (_wei >= 10 ether)
            return safeMul(_wei, 500);
        else
            return safeMul(_wei, 500);
    }
}
```
