<p align="center">
  <img src="logo/9b29cbbe-6739-48ef-b7d2-f2bd8f785470.png" alt="ESNAIL Logo" width="200"/>
</p>

# Ether-Snail

Official repository for the ESNAIL (Ether Snail) ERC-20 token and related contracts.
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract EtherSnail is Ownable, ReentrancyGuard {
    IERC20 public esnailToken;
    uint256 public rate; // ile ESNAIL za 1 ETH
    uint256 public minBuy = 0.001 ether;
    uint256 public maxBuyLimit = 1_000_000 * 10**18;
    uint256 public ownerCut = 28; // 2.8% z 1000
    address public liquidityTarget;
    bool public paused = false;

    uint256 public totalSold;
    uint256 public totalBuyers;
    mapping(address => uint256[]) public tokenRatings;
    mapping(address => bool) public hasBought;
    mapping(address => uint256) public totalPurchased;
    mapping(address => address) public referrerOf;

    event TokensPurchased(address indexed buyer, uint256 ethSpent, uint256 tokensReceived);
    event SalePaused();
    event SaleResumed();
    event RateChanged(uint256 newRate);
    event MinBuyChanged(uint256 newMinBuy);
    event MaxBuyChanged(uint256 newMaxBuy);
    event WithdrawnETH(uint256 amount);
    event WithdrawnTokens(uint256 amount);
    event LiquidityTargetChanged(address newTarget);
    event ReferralRegistered(address indexed user, address indexed referrer);

    constructor(address _esnailToken) Ownable(msg.sender) ReentrancyGuard() {
        require(_esnailToken != address(0), "Nieprawidlowy adres tokena");
        esnailToken = IERC20(_esnailToken);
    }

    modifier whenNotPaused() {
        require(!paused, "Sprzedaz jest zatrzymana");
        _;
    }

    function buyTokens(address referrer) external payable nonReentrant whenNotPaused {
        require(msg.value >= minBuy, "Za mala kwota ETH");

        uint256 tokensToSend = msg.value * rate;
        require(tokensToSend <= maxBuyLimit, "Przekroczono maksymalny limit zakupu");
        require(tokensToSend > 0, "Wyliczono 0 tokenow");
        require(esnailToken.balanceOf(address(this)) >= tokensToSend, "Brak tokenow w kontrakcie");

        if (referrer != address(0) && referrer != msg.sender && referrerOf[msg.sender] == address(0)) {
            referrerOf[msg.sender] = referrer;
            emit ReferralRegistered(msg.sender, referrer);
        }

        uint256 fee = tokensToSend / 100; // 1%
        uint256 buyerAmount = tokensToSend - fee;
        uint256 toRedistribute = (fee * (1000 - ownerCut)) / 1000; // np. 97.2%
        uint256 toOwner = fee - toRedistribute;

        require(esnailToken.transfer(msg.sender, buyerAmount), "Transfer fail");
        if (toRedistribute > 0 && liquidityTarget != address(0)) {
            require(esnailToken.transfer(liquidityTarget, toRedistribute), "Redistribution fail");
        }
        if (toOwner > 0) {
            require(esnailToken.transfer(owner(), toOwner), "Fee transfer fail");
        }

        totalSold += tokensToSend;
        totalPurchased[msg.sender] += tokensToSend;
        if (!hasBought[msg.sender]) {
            hasBought[msg.sender] = true;
            totalBuyers++;
        }

        emit TokensPurchased(msg.sender, msg.value, tokensToSend);
    }

    function pauseSale() external onlyOwner {
        paused = true;
        emit SalePaused();
    }

    function resumeSale() external onlyOwner {
        paused = false;
        emit SaleResumed();
    }

    function withdrawETH() external onlyOwner {
        require(paused, "Zatrzymaj sprzedaz przed wyplata");
        uint256 balance = address(this).balance;
        payable(owner()).transfer(balance);
        emit WithdrawnETH(balance);
    }

    function withdrawUnsoldTokens(uint256 amount) external onlyOwner {
        require(paused, "Zatrzymaj sprzedaz przed wyplata");
        uint256 contractBalance = esnailToken.balanceOf(address(this));
        require(contractBalance >= amount, "Za malo tokenow");
        require(esnailToken.transfer(owner(), amount), "Transfer nieudany");
        emit WithdrawnTokens(amount);
    }

    function setRate(uint256 newRate) external onlyOwner {
        rate = newRate;
        emit RateChanged(newRate);
    }

    function setMinBuy(uint256 newMinBuy) external onlyOwner {
        minBuy = newMinBuy;
        emit MinBuyChanged(newMinBuy);
    }

    function setMaxBuyLimit(uint256 newLimit) external onlyOwner {
        maxBuyLimit = newLimit;
        emit MaxBuyChanged(newLimit);
    }

    function setLiquidityTarget(address newTarget) external onlyOwner {
        liquidityTarget = newTarget;
        emit LiquidityTargetChanged(newTarget);
    }

    function setOwnerCut(uint256 newCut) external onlyOwner {
        require(newCut <= 1000, "Max 100%");
        ownerCut = newCut;
    }

    function rateTokens(uint8 _rating) public nonReentrant {
        require(_rating >= 1 && _rating <= 5, "Rating must be between 1-5");
        tokenRatings[msg.sender].push(_rating);
    }

    function getAverageRating(address userAddress) public view returns (uint256) {
        uint256 total = tokenRatings[userAddress].length;
        if (total == 0) return 0;

        uint256 sum;
        for (uint8 i = 0; i < total; ++i) {
            sum += tokenRatings[userAddress][i];
        }
        return sum / total;
    }

    function getReferral(address user) external view returns (address) {
        return referrerOf[user];
    }

    function getStats(address user) external view returns (uint256 bought, uint256 avgRating, address referrer) {
        bought = totalPurchased[user];
        avgRating = getAverageRating(user);
        referrer = referrerOf[user];
    }
}
