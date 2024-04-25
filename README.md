<p align="center" width="100%">
    <img src="https://developer.vocdoni.io/img/vocdoni_logotype_full_white.svg" />
</p>

<p align="center" width="100%">
    <a href="https://github.com/vocdoni/storage-proofs-eth-go/commits/main/"><img src="https://img.shields.io/github/commit-activity/m/vocdoni/storage-proofs-eth-go" /></a>
    <a href="https://github.com/vocdoni/storage-proofs-eth-go/issues"><img src="https://img.shields.io/github/issues/vocdoni/storage-proofs-eth-go" /></a>
    <a href="https://github.com/vocdoni/storage-proofs-eth-go/actions/workflows/main.yml/"><img src="https://github.com/vocdoni/storage-proofs-eth-go/actions/workflows/main.yml/badge.svg" /></a>
    <a href="https://discord.gg/xFTh8Np2ga"><img src="https://img.shields.io/badge/discord-join%20chat-blue.svg" /></a>
    <a href="https://twitter.com/vocdoni"><img src="https://img.shields.io/twitter/follow/vocdoni.svg?style=social&label=Follow" /></a>
</p>


  <div align="center">
    Vocdoni is the first universally verifiable, censorship-resistant, anonymous, and self-sovereign governance protocol. <br />
    Our main aim is a trustless voting system where anyone can speak their voice and where everything is auditable. <br />
    We are engineering building blocks for a permissionless, private and censorship resistant democracy.
    <br />
    <a href="https://developer.vocdoni.io/"><strong>Explore the developer portal »</strong></a>
    <br />
    <h3>More About Us</h3>
    <a href="https://vocdoni.io">Vocdoni Website</a>
    |
    <a href="https://vocdoni.app">Web Application</a>
    |
    <a href="https://explorer.vote/">Blockchain Explorer</a>
    |
    <a href="https://law.mit.edu/pub/remotevotingintheageofcryptography/release/1">MIT Law Publication</a>
    |
    <a href="https://chat.vocdoni.io">Contact Us</a>
    <br />
    <h3>Key Repositories</h3>
    <a href="https://github.com/vocdoni/vocdoni-node">Vocdoni Node</a>
    |
    <a href="https://github.com/vocdoni/vocdoni-sdk/">Vocdoni SDK</a>
    |
    <a href="https://github.com/vocdoni/ui-components">UI Components</a>
    |
    <a href="https://github.com/vocdoni/ui-scaffold">Application UI</a>
    |
    <a href="https://github.com/vocdoni/census3">Census3</a>
  </div>

# storage-proofs-eth-go

This repository implements ethereum storage proofs in GoLang. 

An ethereum storage proof is a cryptographic proof that verifies a certain token holder holds a certain balance. This is done with the following design:

Each Ethereum account has its own storage trie, which is where all of the contract data lives. A 256-bit hash of the storage trie’s root node is stored as the storageRoot value in the global state trie.

Storage proofs leverage this design by creating a Merkle Proof of this storage trie. The storage proof demonstrates the balance of a token holder for a specific State Root Hash (Ethereum block). This proof can be verified offchain.

![storage trie](https://miro.medium.com/max/529/1*f0vOn0lRgrY5NjFlbUMBFg.jpeg)

This is possible due to [EIP1186](https://github.com/ethereum/EIPs/issues/1186) which introduced the web3 call `eth_getProof` which is a convenient method for obtaining the storage proof.

More info regarding storage proofs:

- [https://medium.com/hackernoon/getting-deep-into-ethereum-how-data-is-stored-in-ethereum-e3f669d96033](https://medium.com/hackernoon/getting-deep-into-ethereum-how-data-is-stored-in-ethereum-e3f669d96033)

This library was designed to allow the generation of on-chain token-based censuses for off-chain voting with Vocdoni. The best place to learn about using storage proofs to create a census is the [developer portal](https://developer.vocdoni.io/protocol/census/on-chain#erc-20-token-storage-proofs).

### Table of Contents
- [Status](#status)
- [Reference](#reference)
- [Getting Started](#getting-started)
- [Examples](#examples)
- [Preview](#preview)
- [Disclaimer](#disclaimer)
- [Contributing](#contributing)
- [License](#license)


## Status

**This GoLang module is WIP.**

TODO list:
- [x] create an interface around token 
- [x] add support for MiniMe tokens
- [ ] add support for EIP721
- [ ] explore adding support for other token standard rather than ERC20
- [ ] write tests


## Getting Started

This repository provides all of the golang packages required to generate and verify an Ethereum storage proof for an ERC20 contract. To use this library, just set up a working golang project and import the `token` and `ethstorageproof` packages.

## Reference

Automated documentation of the packages in this library is provided at https://pkg.go.dev/github.com/vocdoni/storage-proofs-eth-go

## Examples

The following tutorial provides an example for integrating ethereum storage proofs into your project.

### Initialize a `token`

Create a `token.ERC20Token{}` type variable, and initialize it with contract `0xdac17f958d2ee523a2206206994597c13d831ec7` (Tether Stablecoin).
```golang
    web3 := "https://web3.dappnode.net" // do not abuse please
    ts := token.ERC20Token{}
    ts.Init(context.Background(), web3, "0xdac17f958d2ee523a2206206994597c13d831ec7")
```

Fetch the basic token data (decimals, name, etc.) and the balance for the token holder `0x5041ed759dd4afc3a72b8192c143f72f4724081a`.
```golang
	tokenData, err := td.GetTokenData()
	if err != nil {
		panic(err)
	}
	holderAddr := common.HexToAddress("0x5041ed759dd4afc3a72b8192c143f72f4724081a")

	balance, err := td.Balance(holderAddr)
	if err != nil {
		panic(err)
	}
	fmt.Printf("balance from ERC20 ABI call balanceOf() is %s\n", balance.String())
```
### Index Slot
For each contract we need to find the **storage index slot**. This value depends on the contract implementation and signifies the storage position in which the token balance is stored.
For a map-based balances, `map(address)=>uint256`, the storage slot for a specific token holder will be equal to `keccack256( tokenHolder + indexSlot )`.
```golang
	tk, err := token.NewToken(token.TokenTypeMapbased, contract, web3)
	if err != nil {
		log.Fatal(err)
	}
	slot, balance, err := tk.DiscoverSlot(common.HexToAddress(holders[0]))
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("index slot for the contract is %d\n", slot)
	fmt.Printf("balance found on the EVM storage is %s\n"), balance.String())
```

### Fetch Storage Proof
Now lets get the storage proof for the previous token holder on the last Ethereum block, this is obtained using EIP1186 web3 method `eth_getProof` for the last Ethereum block.
```golang
	sproof, err := tk.GetProof(holderAddr, nil)
	if err != nil {
		panic(err)
	}
```

### Verify Storage Proof
Finally, for a key, a value (balance) and a Storage Hash Merkle Root, verify the Merkle Storage Proof is valid.
```golang
	if pv, err := ethstorageproof.VerifyEIP1186(sproof); pv {
		fmt.Println("account proof and storage proofs are valid!")
	} else {
		fmt.Printf("account proof and storage proofs are invalid (err %s)\n", err)
	}
```

Proofs of **non existing values** can also be generated and verified using the same procedure with the only difference being the proven value is equal to `0x0`.

--- 

Example of a proof obtained with the `eth_getProof` method:

```json
{
   "address":"0xdac17f958d2ee523a2206206994597c13d831ec7",
   "accountProof":[
      "0xf90211a0384dea0e04cdf33c7972d6fda16170b9b04bb0b7f19907e1376343cb46757225a0a767ccd497f2515220a3f5667fe738f4ee669792c380a1894aec4be0ece5aa81a09713acc380ec317fdc5b827dc23c2f63f0a4b9923242c99bee7f186e47545ceea02e80a0af7fa0445833fd58b3240545d9b17b844a5715369e4ec0c76751a6aa27a05cbf7446065dae784d8275ab38a91567a63dc38b6c6ab47f34fbc27b112511aaa0252f860b05658694d03e82122b2b53c0a2aa27b3cdd6c6b506f6ed78611b8ff0a04c0ac4a26e60b425a8d5f93202334efedd06ff3277e5619ea69e9bcba8d2808ba0a30e5359136d5f00f5f4742aadf0c892e7ad5e3e4eafb4d0ff81e47b68b629dea09d21f0a6ffa5ca1505a6f441bbff23d00cd2d7e4eb3e53d2cad40925e898cd21a036b087a125f6731a255962bb1e3d968b9b17cf3906fc4165b8c0358c60d55dbfa05c5a1d68f48766dc9759fd68c53037c772c46bfc7c17d8a436cd94bbe713b295a0dec84fd1a29839d1f1753c6bea4411bb52ed321a6e635bb73feecd784a884d41a0e2c512fed45b79beda8d69490b125a491319d96de26b66dafcc402fb4f924d33a06c3ba8686212011a60318df596eea333836ec0d1b6f6c19c2bcea94370fda9bba0f13806aec144b2f2686c6097945610e127b35449021409425b43cb01699db911a04cf068b6dbd041b1ac3417262503cdd7370a100f98d202a918888d610b825eec80",
      "0xf90211a017fc8a237f709a9d525259665f6c44fe95ac2c1563a1275f8d6f040a6e6bd712a064e92c041d4010a9fc8184cf0d8ce5ec35b29a6f6eb9d28aa5d261e831b0a247a06803e1c46d87cfac1b18db2a887829759847ebfdf9ff536da170b6883e890f2ca001c238a230844b25f75c7fce273e51030d4a68faaecc90cc0e0cbcf75e990b02a08dcd9431d8553e35753a4059d6aaceda195714a601fbb1fe4eb82b3291175c71a0b5ddf286be094782da6a3f35c2a1fc2dc9d5daa542e02343f85f5b611cc38d35a039d1b06d8f2df8f4497ffd12ed032a37cb776463e9b9bd6dc58da780c6c4bcd4a054fb4472575537badd15ef021be027f7726d761ccb3088da4014bed1ccd0d936a0bde473565f25d95ba83355ef61179db0e5f09b167ecde2c3b2ff38af387547d1a048253d7af577733f9d77146f25e1939a6dc9b1b7b4df7e92a5377e0c21bad6cea05cbf482b66d59074e021ea08582a5c34ff360f2a548238541da819779f27f09ea09acc6af56b43137efafe6caee4923a50e531a026597d5b93bde87b74389ace96a068ea1192ecf9f0c2a8b5e40bebc76215f87b5fc9f3f9958eaea7fe64b34da398a082c17e65c7e205247575cc0a543ca5d43da2bdfbfd68de21abb7aa32267c31ada0686aa8f993fb794a2bdf469b4196c8fd31603d9012f16287a06106dfe5874d3ba0e78109090bca2991159eb726ed0b125a5542b6249563a339a2fc1c8b6fa1889d80",
      "0xf90211a0ac9b57ddb96081b19763e007c420b9166ea3e3cb81a92ec6779fa5012c4a00cea0a2f3d95f6f878ed41c2211e079afabecbb40fbbd382714bfc63c032ee3499f24a0925708c42cda17afe109862da1be30f4531fea0c57b160331bfe91893b3b71fba02ef945d4edc9de0a76e0901ebd61398672a7bd422a6fd5840d7be4be5c65a232a068c67517f4aab7a038aae55a15e1513009f7919305d086b413fb0e801be943f0a05e8613d39e40c546aad1961961ce4dc04387bac1a77b46b065779604f0e41606a062cbff30606d51cd45f458ce61e8ada839895a7a747ea261db30eea7a6be330ca00958eab204a900567e44cf47298dc3cb34333a3e45d657674e54f8625c578662a06257c48f08e7526274385077b00d87d99fc210a703c90fe35def31bcc11cefc3a02abc60edeed5399cfb676ebe7426a57c85b5e018eab10812f89a5b16c76cea68a0ffba8197099393d36377cc4b9a5a1346c925e2a6d1fc14e9ca56f5795102db12a0e9384096699720f9db7e5e66ff0c65874033cd8aa26f5f4b2841c44308e17117a0b30af72e6077c838b9e1dff5fe711d3a75759d22224258a3846449940bf545daa0c2eb8df2bd709ad10e4051155079b277ed20b33e9781d45451ebbc968d8e3c4ba0aafed1e2706d2f38feb683ff3c93bf264b9edddae4716e985a0844c2e3380899a08ae727a23403c25e37135e9a180be1a34c2ebe9981f59929fdc65f98837d003a80",
      "0xf90211a0e1bbf50f7cee3f3da069c14ce664ed374f53951dc21696972eee12c1dc50839da0b6e6c5c3078a1ccb7e08d2ffadd1a000bb6a84fd70d32be60ea780da5f4c15f9a07235156a477b1dc99dfb801a73af75d6ecca8d8500ae645aaabcd4f64fd0fa2ea041c4133ef628f5698f17676e491c34355a3c345c4092108f3eba73eb145459bda0709f062e88f4a2e47a5eb6c7e10bbcf7a64891bb0b924d8f860673f7b44f1bbfa05b1b503e5f6bd3115314e767ced96aed64ff2a4a83b801f34846b87985ca3155a02da2e53cc3a97780e98fe16433e0976f7ae7268531b7e5ee9141c790812e200ca06016c3678dc803061ac5de8c0a21a994c4f98f2e7717f5ef26c1d1ad7afc6af5a09cdfb998edffbcdb10424b6079fa49c6474f1786ebd6aa3d53a868ee94868f42a0669eb33ebf4ef9420273f1464b946b893758ccdb4742da5385d22aa76a896291a015e29157a0242ffd57a2895cb5d72c00321a418cf51d8ec16c69d529e4504aaea0678382e1bd36c2318a4244b8ac38225c065c77c024a262aa5d6bd667acc529f0a081d7411b485a5288436cc91da7838f10fb5c4229677ffc7dfb59473e5dbc4e5ca043937cb4b99b1984b10d3167814e460a05de339f06c19755d440df7bda40fe57a006d96232e235fc308b5bb302a69793cd38738df18864f0d1671bb53d48d3841ba0cec792dc460edfd37e647c560c53f2edb85d978bf4c254e512b823f7e890db8880",
      "0xf90211a0fd4dc179175e5af393ab1d7551bd3a9c5760ed9679df54d34cce97daca2949aaa03baaa57f4b7492162639d4154d45ae5792c2b58065c37fd87b10b75ce897fd80a097474dfc4ee2d38407026dc70abfc9a74ba999328ccd3cd0715f5f2f58dcb154a02f8effca6153706f073550808745fcb5c2333e2374ab1aab0f0c5b7616aebc5aa0067034da9840d6ce9bc8bed341805b38d2f66930278cb0d106cf49b8a353157ca070e069dfffb9e1281493c929825673cf27eeddd39b139561cda7afb4a09c9726a08eb7abd6b39e4ec53b3a63d16c84f98bd2e54a6443078a76386851d8c660aceca0481ca3da2b45dcd27087f4dce07a5776dd7f041b06daf586929fb6a184697472a0ec394b987b3e586e27bc33dbda68b68230abd5a6bb43bd3274eb83b6f88fc6b8a058cbedf165bc9886ce3f3358b276c6d64fe8326743d09e29927e64b915b96bcba0a8ce82444f7b90bd648ed79a07f5473de5ac574f226dc2837398c79c22403397a00ef90fd591599ffd78bb63d02688692c6b8290d63b8d3d6a05ec3e914ff13d87a010fbb6a665a92bd47ecda2f2f7f0f6b0bcec460e0bfee79639564de05c35c65aa0a0d80741d491df5b1703b29fe2ff7b7cf1688a96ae9f82ea4b7ad14f075dd65ca089a0d75785af96ded82b990c18a055c33b6de178dca530eb4f6d686327e62dfba003f6fc685410b06341c89ebe59b5d334c2b592b105da3456e120fa5946772a0080",
      "0xf90211a0667022a2e9d08e61ab0e84eba409a3faa1f117b1448cab10b08a4b3157c12779a0f060906007684bba15aca67b8b8de5d52359d79ee8e0d5427c9816cc1f89b403a09770f1dc4fcca0eb269cbb9065375d88f79d57581d0d2928e8565c9cac03ffb1a0c02c1caec46022380688a901ff39f44a69b24b6f5e8c7423bdc81ab7a0842ae5a03ef6974fb165ccedc4d1272e8cef18fc7c7dd158051fa4fcbe50cf52309289e9a0ea722726bf8c8d8a9351a1d4e530c428f582ab6cab8cc9e33ead52983898cb1ea09fd533ae1eba26257907b7e5ddd6bf710ca5b08ab066cdaeb80c805a44727543a0cfb0452dd7d9a5ac8f1a5ac7c33105285117a80bfa9ffaea4ef232126dbc2f31a0ab9112bb96477a58a2245426f4a4e79a655bb1faace978a3719883dc9c67f347a0e836d2a68e3dc58f9cb0dc770a2aa9321e6beb5747c36ca1aa28796ec151b0cea0a13881e2f4a05e8aa92f20077efe09186164dee35e20ba1d0bfc7c7b26a92e70a0945f49ad82c981890faae5f2551ed7d6d7dce4eddd2bb0bedf2119fa731bf14ba09b28663cea3b6c37a8e07e1a588a381cc3001a11f21add5cc038a17a4397d30ea04dff355675e2f31c9c2de55d16b9f8ca53382f5c024213a5b7936acfb54656cba061a0bb68122afdfae0fb4f556dc3a65d79a690f0e1918004a93f996467ddd550a04b767f6b38c7a6439a4b7b6066207638b9656c416f9ad09a3ee52c600e681c4780",
      "0xf8f18080a0149f0c3b727be267847fe12d583df188a307bc85b5adeb7577450367ef875894a00b3a26a05b5494fb3ff6f0b3897688a5581066b20b07ebab9252d169d928717f80a01e2a1ed3d1572b872bbf09ee44d2ed737da31f01de3c0f4b4e1f0467400664618080a01fb693867275623bcbc46e13db58ddcd0ceda37cd8c5d39ebc56d15b535d40fca07aad8ea34d91339abdfdc55b0d5e0aa4fc3c506f56fd2518b6f8c7c5d2ed254880a0e9864fdfaf3693b2602f56cd938ccd494b8634b1f91800ef02203a3609ca4c21a05904860db3383d925f9a36578fcea05e0ef02ee49d21ce50d0df7056df00b0f880808080",
      "0xf8669d3802a763f7db875346d03fbf86f137de55814b191c069e721f47474733b846f8440101a03310913fe74cfb66dbde8fe8557b48e8e65617a17c2375a581c32d49f812cde4a0b44fb4e949d0f78f87f79ee46428f23a2a5713ce6fc6e0beb3dda78c2ac1ea55"
   ],
   "balance":"0x1",
   "codeHash":"0xb44fb4e949d0f78f87f79ee46428f23a2a5713ce6fc6e0beb3dda78c2ac1ea55",
   "nonce":"0x1",
   "storageHash":"0x3310913fe74cfb66dbde8fe8557b48e8e65617a17c2375a581c32d49f812cde4",
   "storageProof":[
      {
         "key":"95eea00c49d14a895954837cd876ffa8cfad96cbaacc40fc31d6df2c902528a8",
         "value":"0xd647b234389e",
         "proof":[
            "0xf90211a03697534056039e03300557bd69fe16e18ce4a6ccd5522db4dfa97dfe1fad3d3aa0b1bf1f230b98b9034738d599177ae817c08143b9395a47f300636b0dd2fb3c5ea0aa04a4966751d4c50063fe13a96a6c7924f665819733f556849b5eb9fa1d6839a0e162e080d1c12c59dc984fb2246d8ad61209264bee40d3fdd07c4ea4ff411b6aa0e5c3f2dde71bf303423f34674748567dcdf8379129653b8213f698468738d492a068a3e3059b6e7115a055a7874f81c5a1e84ddc1967527973f8c78cd86a1c9f8fa0d734bd63b7be8e8471091b792f5bbcbc7b0ce582f6d985b7a15a3c0155242c56a00143c06f57a65c8485dbae750aa51df5dff1bf7bdf28060129a20de9e51364eda07b416f79b3f4e39d0159efff351009d44002d9e83530fb5a5778eb55f5f4432ca036706b52196fa0b73feb2e7ff8f1379c7176d427dd44ad63c7b65e66693904a1a0fd6c8b815e2769ce379a20eaccdba1f145fb11f77c280553f15ee4f1ee135375a02f5233009f082177e5ed2bfa6e180bf1a7310e6bc3c079cb85a4ac6fee4ae379a03f07f1bb33fa26ebd772fa874914dc7a08581095e5159fdcf9221be6cbeb6648a097557eec1ac08c3bfe45ce8e34cd329164a33928ac83fef1009656536ef6907fa028196bfb31aa7f14a0a8000b00b0aa5d09450c32d537e45eebee70b14313ff1ca0126ce265ca7bbb0e0b01f068d1edef1544cbeb2f048c99829713c18d7abc049a80",
            "0xf90211a01f7c019858f447dbff8ed8e4329e88600bab8f17fec9594c664e25acc95da3dba00916866833439bc250b5e08edc2b7c041634ca3b33a013f79eb299a3e33a056fa0a8fae921061f3bc81154b5d2c149d3860a3cdef00d02173ee3b5837de6b26f56a097583621a54e74994619a97cf82f823005a35ad1bf4795047726619eccd11ad4a0be39789c4abdb2a185cce40f2c77575eee2a1eab38d3168395560da15307614aa0c46a7fcea5501656d70508178f0731460edcce1c01e32ed08c1468f2593db277a0460f030038b09b8461d834ded79d77fd3ed25c4e248775752bf1c830a530ef2ba03d887abec623c6b4d93be75d1608dcacb291bf30406fbbc944a94aa4203fb1eba0475eeb07d471044af74313093cfbb0201e17a405d7af13a7ae6b245e18915515a0ea794035230f90e14f4f601d0e9f04217ed02710df363417bc3dd7dda5a84608a08aa9f0c44e5a9e359b65d4f0e40937662f07af10642a626e19ca7bc56c329706a05677c9b342c1cbcfd491cd491800d44423d1bdc08ab00eb59376a7118f7c4e23a0b761f22d67328e6c90caf8be65affb701b045f4f1f581472fc5f724e0c61328da0b6727240643009e59bba78249918a83ca8faeaf1d5b47fe0b41c356e8ae300dda0d7cbeb12faf439126bdb86f94b6262c3a70e88e4d8a47b49735f0b9d632a5df8a0428fe930556ce5ea94bcc4d092fe0f05a2a9b360175857711729ab3a7092a81780",
            "0xf90211a0a81ff74945250ba9925753e379aa8815a3fe77926aad2d02db4d78a3da7ecb48a0e795c64d2b738b34ebb77a3307df800c9f1fe324b442cd71ffa5fd268ec12ffca020a5f968a1c8292d08135cb451daed999a44eb0cdd04526a89c38e753398c50ca0cc7165f6b984f2f2569d101d70f72af94eb79b18c126895da2d3cf557b5aca51a00a10f4dc5851e71f195cca5bd7a2a62a6c3ca03d95e91c7dc55e55f4cb726903a09ecaaf877b18a55ca4001fe46cc10389c94104abc2994cdfe5843f28814b119ca099f59b2b52ee9d9b44ab7123a86775dcfe6c50301dd7cf9c6fc6ea968c1d2a01a0fccda5c1489dd3268fcf2471f6a372fc3afe5eebecc0ba9fc3c023d85da26aeda0d730ea7aabdad2e5451e826726e53c86e6e805e46220bc7edd80bf3f7e467f96a074f3d767a84557aa9559b08b2d1f93d8205827c042d1ac616b8cf37e22de2beca03ca1fe479ea1e64c3bfeae9c603ef4f55c270de0bb4c79d045f67d3481ca2852a0bdcbc3d154db40b5faa1f6875f46d485b96bdfffb4513da631f9b302768faae7a096923f4f559c7b7f912587292cc865aba5934e9e75c9aa88f20de73e928c6c51a0f52ca9710977327dd407ba9d9f00b0075d7e312ee19686f1b53579585ff50132a076adb4cf98f9af261d1b2a147b9b2cbace1647eab537ca1a55e8d40d35775b74a0723fa2f72d6a983939bc9bf9ee22e05fc142437726450e5707c60155bc34ec0580",
            "0xf90211a039a64fd9b31f3ea3bf9a991ca828eadb67cf9ae0ebbcdd195d297454699580e8a0c95d85a63beb9a02b56d032116d86fdf9a64065dd5d44e33acef674ae3ceb6f9a0d83ef07a99302abccdce1289d353a20494713f45d8edbd3c2e87f08788a878d9a063a19aee41fec40a98edcf4d60c759c509b512bfc5d9feae7de50c8c2d00eee7a029ce5c8c1ac9939cbf481274b8e6f5f24a430136dc5aab7deb489a9ce7db5a95a03b45c53d2e4f54e49a53eb298aee828f4803581ee60ca52940533b77c3e3fadfa082b8084dfc0337a49fd5d53ce107fa0bdcdf25ac7bbac017d7ac250b77c8764da0d4dab90004fd3bb36b2fa5f6e914a7c90820d119e5ba3ba8439270c6cfbd0a18a0bd9286c9248ae8a953d00aa9906f06b6f0364d0bb9fb615de04c9f8d5b5fd346a00955eccaa41a17fafb0ed66272b5183b3a30a973af7a43e0f25f0b640fd5df0ca0a7ba773602ace05991211770fc5555dd9c55e2518bcb005d1db022584a89132da0450b066588b44701992ab4a926d5dd185dd465abf1e51a3f60ab4a9f924cf85aa05eda0687636512339d0db99eade61c0d44358bb8027185296ad59b4cedd2935aa0cd18604dee296e5443b598e470a04efde04fbb0213c0fda8cc3af20dae2a1c34a01c0474c1bf15e4732d1c22a1303573c7ec8643f711354e853ab26038b2eebe25a081a0b2d329a9519375f71384a7130faafc3a1379db59bfb7db7135c9d27d9cc080",
            "0xf901f1a0e14d9caa85464966f0371f427250e7bf2d86f6d41535b08ea79044391d6f0fe7a08bea233c92e4c1d05bd03d62cd93974cc29f6df54e6fa2574bf8dfb3af936a85a0dd4f4af9ab72d1bbbf59c286a731614f48bf3cdcef572cd819426fc2a30ae5d9a058465ea8e97b2f3a873646f9504aff111ab967abaea8ebb2fe91bf058ea162f3a06466c41eb770bae5c07c26cd692540e7f9af70fee2bc164e12f29083ebd2cca1a0fe0925da033ffb967ca1e1d6505c2ca740553916c3b9a205131d719b7921fb00a0973ecee958d1305f1b6f8159a9732e47f5fbd4121f60254ea44a8f028308150780a0f81717c7f8702ed39d89a34fc86827aab31db26b22d22ee399667e4a44081c95a0e276ec24ee74ee61bdbe92e8afa57813d59f26e0ad34f24d71f0627719f8a11ea00a8ec2f6922480890f9e67c62c1654dcccf1710a01a378f614e02a3785d42d7ea08c43cc0c6c690dd87cf42691e9ead799c5ddbcfc9458ebd054a46acecedbe9f7a0d3727ed077c60ed6ff1d9d5d20fdf2912a513942d0efee9ec2d97d33ab9b7f56a03f293b0c0f25b9aa2735c4e42b132008d8af2ecfaaefdc940dc94cf312f5a1dda04860bc5f19829bb277554927c5b6dbc0af93025aca49aed3be020a3080996a26a0e47475bf4f62364d8a0dd87f2c21d0b64ef4d9aaf5fc46d9b1e3af5b83f56d4580",
            "0xf89180a0dd3524693059f48ce47c5dd82d0462953a5141c7abc5a63981637b71e0100bcf80a033ca98826ea65c1c4be16e1df12e78142d992bf1fc0189401632ceff3fcbbe7880a0e5daf803b0890d164e287fa43b51332286cddeb9b62280426037fc02dc2b4d5f8080a0bfa907c2e30720a07347d23cfa573dec8c9b3afa51a4a0e0783a481da0fcb1b38080808080808080",
            "0xe79e208246cec5810061f4ff7efe1dcd6cb407d59abc3478830df04484584c868786d647b234389e"
         ]
      }
   ]
}
```


## Disclaimer

The code in this repo is WIP. Please beware that it can be broken at any time if the release is `alpha` or `beta`. We encourage you to review this repository and the developer portal for any changes.

## Contributing 

While we welcome contributions from the community, we do not track all of our issues on Github and we may not have the resources to onboard developers and review complex pull requests. That being said, there are multiple ways you can get involved with the project. 

Please review our [development guidelines](https://developer.vocdoni.io/development-guidelines).

## License

This repository is licensed under the [GNU Affero General Public License v3.0.](./LICENSE)


    Vocdoni Ethereum Storage Proofs
    Copyright (C) 2024 Vocdoni Association

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU Affero General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU Affero General Public License for more details.

    You should have received a copy of the GNU Affero General Public License
    along with this program.  If not, see <https://www.gnu.org/licenses/>.

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL%20v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
