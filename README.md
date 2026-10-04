// SPDX-License-Identifier: MIT
 ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract WhitelistNFT is ERC721, Ownable {
    bytes32 public merkleRoot;

    uint256 public immutable maxSupply;
    uint256 public mintPrice;
    uint256 public totalMinted;

    mapping(address => bool) public claimed;

    event Minted(address indexed user, uint256 indexed tokenId);
    event MerkleRootUpdated(bytes32 newRoot);

    constructor(
        bytes32 root,
        uint256 supply,
        uint256 price
    )
        ERC721("Whitelist NFT", "WLNFT")
        Ownable(msg.sender)
    {
        require(root != bytes32(0), "Invalid root");
        require(supply > 0, "Invalid supply");

        merkleRoot = root;
        maxSupply = supply;
        mintPrice = price;
    }

    function mint(bytes32[] calldata proof) external payable {
        require(!claimed[msg.sender], "Already claimed");
        require(totalMinted < maxSupply, "Sold out");
        require(msg.value == mintPrice, "Incorrect ETH");

        bytes32 leaf = keccak256(
            abi.encodePacked(msg.sender)
        );

        require(
            MerkleProof.verify(proof, merkleRoot, leaf),
            "Not whitelisted"
        );

        claimed[msg.sender] = true;

        uint256 tokenId = ++totalMinted;

        _safeMint(msg.sender, tokenId);

        emit Minted(msg.sender, tokenId);
    }

    function setMerkleRoot(
        bytes32 newRoot
    ) external onlyOwner {
        require(newRoot != bytes32(0), "Invalid root");

        merkleRoot = newRoot;

        emit MerkleRootUpdated(newRoot);
    }

    function setMintPrice(
        uint256 newPrice
    ) external onlyOwner {
        mintPrice = newPrice;
    }

    function withdraw(
        address payable recipient
    ) external onlyOwner {
        require(recipient != address(0), "Invalid recipient");

        uint256 balance = address(this).balance;

        (bool success, ) = recipient.call{
            value: balance
        }("");

        require(success, "Withdraw failed");
    }
}
