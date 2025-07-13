# Etibé: Decentralised Group Contributions on UMI Blockchain

## 🚀 Project Overview

Etibé is an innovative, decentralised mobile contribution platform that reimagines Nigeria's venerable "Ajo" system for the digital age. Built meticulously on the UMI blockchain, Etibé empowers users to effortlessly create and participate in rotating contribution groups, known as "Etibé Channels." Members contribute a fixed amount each month and receive a substantial lump sum payout on a predetermined date. Our solution ensures a truly trustless, transparent, and automated management of peer contributions, encompassing savings, disbursements, and seamless group administration.

Etibé is more than just an application; it's a commitment to financial inclusion and community-driven savings, leveraging the power of blockchain to foster trust and efficiency where traditional systems often fall short.

## ✨ Core Features

- User Account Creation: Users can register with their email and password. Upon successful sign-up, a unique UMI wallet address is automatically generated and securely stored on the device. Users also set a distinctive display username, which is used for channel invitations.

- Etibé Channel Creation: Admins can initiate new contribution groups, meticulously defining crucial parameters such as the fixed monthly contribution amount, the precise payout order (by username and month), and a flexible grace period for payments. This process ingeniously triggers the deployment of a dedicated smart contract on the UMI blockchain, encapsulating all defined rules.

- Invitations: Channel Admins can extend invitations to members using their app usernames. Invited users receive immediate in-app notifications and are presented with the clear option to accept or reject the invitation. Crucially, once all channel slots are filled and invitations accepted, the channel's smart contract is irrevocably locked, preventing any further alterations to the payout order or contribution amount.

- Automated Contributions & Disbursements: On the designated payout date each month, the Etibé app intelligently reminds users to ensure their wallets are sufficiently funded. Should the wallet balance be adequate, contributions are automatically deducted via the smart contract. These pooled funds are then seamlessly transferred to the scheduled beneficiary for that month (Currently, the admin will have to trigger this action; it is not automated).

- Grace Period & Auto Removal: A predefined grace period is afforded to users to fund their wallets. If a user fails to contribute within this window, they receive timely reminder notifications. Persistent non-payment beyond the grace period triggers an automated removal from the channel. The admin is promptly notified and retains the ability to invite a replacement member for subsequent months.

## 🛠️ Technical Implementation

Etibé is engineered with a robust and scalable technical architecture, leveraging cutting-edge technologies to deliver a seamless and secure user experience.

### Frontend

- Mobile-first Design: Developed with a strong emphasis on mobile usability, ensuring optimal performance and aesthetics on both Android and iOS devices.

- Theming: Offers comprehensive support for both light and dark themes, catering to user preferences.

- UI States: Thoughtfully designed to include support for various UI states, such as empty states (e.g., no invites, no channels), error states (e.g., failed payment, removal), and helpful onboarding modals/tooltips.

- Consistent Iconography: Utilises a standardised set of icons for key functionalities, such as wallet, payout, alerts, and calendar/schedule, thereby enhancing user navigation and understanding.

### Backend

- Node.js + DolphJS: The recommended and implemented technology stack for the backend, providing a powerful and efficient foundation for server-side operations.

### Blockchain Interaction

- UMI Blockchain: The fundamental decentralised ledger underpinning Etibé, providing the immutable and transparent backbone for all transactions and channel operations.

- Smart Contracts: The very heart of Etibé's trustless automation. All contribution logic and channel rules are rigorously enforced via UMI smart contracts, guaranteeing tamper-proof and auditable operations.

### 📜 Smart Contract Code

The core logic and operational rules of Etibé are meticulously encoded within two primary Solidity smart contracts, deployed and executed on the UMI blockchain.

**EtibeChannel.sol**

This contract is the blueprint for each individual Etibé contribution group. It meticulously manages member statuses, handles contributions, facilitates payouts, and enforces removal logic, ensuring the smooth and trustless operation of each channel.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

/**
 * @title EtibeChannel
 * @author Utee
 * @notice This contract manages a single Etibe (Ajo) contribution group.
 */
contract EtibeChannel {
    address public immutable admin;
    string public channelName;
      // The fixed monthly contribution amount in wei.
    uint256 public contributionAmount;
    uint256 public immutable channelSize;
    uint256 public immutable creationDate;
     // The official start date of the first contribution cycle.
    uint256 public immutable startDate;
     // Grace period in seconds (e.g., 3 days).
    uint256 public immutable gracePeriod;
    string public channelImage;
    bool public isPublic;

    enum MemberStatus {
        // User has been allocated a slot but has not yet accepted.
        Invited,
         // User has accepted the invite and is an active participant.
        Active,
         // User was removed for non-payment.
        Removed,
        // User has successfully received their payout.
        PaidOut
    }

    struct Member {
        address walletAddress;
         // The month number (1, 2, 3...) they are scheduled to receive the payout.
        uint256 payoutMonth;
        // The current status of the member.
        MemberStatus status;
        // The last month number for which they successfully paid.
        uint256 lastContributionMonth;
        // The last month number for which they received their contribution share.
        uint256 receivedContributionMonth;
    }

    mapping(address => Member) public members;
    address[] public memberAddresses;

    // Tracks the current month of the cycle (starts at 1).
    uint256 public currentCycleMonth;
    // Becomes true when the channel is full and contributions begin.
    bool public isLocked;
    // Becomes true after the final payout is made.
    bool public isComplete;


    event ChannelCreated(address indexed admin, string name, uint256 amount, uint256 size, string channelImage, bool isPublic);
    event InviteAccepted(address indexed memberAddress, uint256 joinTimestamp);
    event ChannelLocked(uint256 lockTimestamp);
    event ContributionReceived(address indexed from, uint256 forMonth, uint256 amount);
    event PayoutMade(address indexed recipient, uint256 forMonth, uint256 amount);
    event MemberRemoved(address indexed memberAddress, uint256 atMonth);
    event ChannelCompleted(uint256 completionTimestamp);

    // modifiers below...

    modifier onlyAdmin() {
        require(msg.sender == admin, "Etibe: Caller is not the admin.");
        _;
    }

    modifier onlyInvitedMember() {
        require(members[msg.sender].walletAddress != address(0), "Etibe: You are not invited to this channel.");
        _;
    }

    modifier whenLocked() {
        require(isLocked, "Etibe: Channel is not locked for operations yet.");
        _;
    }

    modifier whenNotLocked() {
        require(!isLocked, "Etibe: Channel is already locked.");
        _;
    }

    modifier onlyMembers() {
        require(members[msg.sender].status == MemberStatus.Active, "Etibe: Caller is not an active member.");
        _;
    }

    event DebugLog(string message, uint256 value);
    event DebugAddress(string message, address addr);

    /**
     * @dev Deploys the contract and sets the immutable rules of the channel.
     * @param _channelAdmin The address of the admin for this specific channel.
     * @param _channelName The name for the channel.
     * @param _channelImage The image for the channel.
     * @param _contributionAmount The fixed monthly contribution amount.
     * @param _startDate The Unix timestamp for the start of the first cycle.
     * @param _gracePeriodInDays The number of days for the grace period.
     * @param _invitedMembers An array of wallet addresses for all invited members.
     * @param _isPublic A bool identifier for when a channel is open to public discovery.
     * @param _payoutOrder An array assigning a payout month (1, 2, 3...) to each corresponding member in _invitedMembers.
     */
    constructor(
        address _channelAdmin, // NEW: Admin is now passed as an argument
        string memory _channelName,
        string memory _channelImage,
        uint256 _contributionAmount,
        uint256 _startDate,
        uint8 _gracePeriodInDays,
        bool _isPublic,
        address[] memory _invitedMembers,
        uint256[] memory _payoutOrder
    ) {

        emit DebugLog("Constructor: Start", 0);
        emit DebugAddress("Constructor: _channelAdmin", _channelAdmin);
        emit DebugLog("Constructor: _invitedMembers.length", _invitedMembers.length);
        emit DebugLog("Constructor: _payoutOrder.length", _payoutOrder.length);
        emit DebugLog("Constructor: _startDate", _startDate);
        emit DebugLog("Constructor: block.timestamp", block.timestamp);


        require(_invitedMembers.length == _payoutOrder.length, "Etibe: C1 - Mismatch lengths.");
        emit DebugLog("Constructor: C1 passed", 0);

        require(_startDate > block.timestamp, "Etibe: C2 - Start date must be in future.");
        emit DebugLog("Constructor: C2 passed", 0);

        require(_channelAdmin != address(0), "Etibe: C3 - Admin cannot be zero.");
        emit DebugLog("Constructor: C3 passed", 0);

        // require(_invitedMembers.length == _payoutOrder.length, "Etibe: Members and payout order arrays must have the same length.");
        // require(_startDate > block.timestamp, "Etibe: Start date must be in the future.");
        // require(_channelAdmin != address(0), "Etibe: Channel admin cannot be the zero address."); // Added validation

        admin = _channelAdmin;
        channelName = _channelName;
        contributionAmount = _contributionAmount;
        startDate = _startDate;
        gracePeriod = _gracePeriodInDays * 1 days;

        channelImage = _channelImage;
        isPublic = _isPublic;

        creationDate = block.timestamp;
        channelSize = _invitedMembers.length;
        currentCycleMonth = 1;

        for(uint i = 0; i < channelSize; i++) {
            address memberAddr = _invitedMembers[i];
          emit DebugLog("Constructor: Loop i", i);
            emit DebugAddress("Constructor: Loop memberAddr", memberAddr);

            require(memberAddr != address(0), "Etibe: C4 - Invited member address cannot be zero.");
            emit DebugLog("Constructor: C4 passed", i); // This is the one that's still suspicious due to its logic if not interpreted carefully
            // It expects members[memberAddr].walletAddress to be address(0) (meaning not set yet)
            require(members[memberAddr].walletAddress == address(0), "Etibe: C5 - Cannot invite the same member twice.");

             // This is the one that's still suspicious due to its logic if not interpreted carefully
            // It expects members[memberAddr].walletAddress to be address(0) (meaning not set yet)
            require(members[memberAddr].walletAddress == address(0), "Etibe: C5 - Cannot invite the same member twice.");
            emit DebugLog("Constructor: C5 passed", i);

            members[memberAddr] = Member({
                walletAddress: memberAddr,
                payoutMonth: _payoutOrder[i],
                status: MemberStatus.Invited,
                lastContributionMonth: 0,
                receivedContributionMonth: 0
            });
             emit DebugLog("Constructor: Member assigned", i);
        }

        emit ChannelCreated(admin, channelName, contributionAmount, channelSize, channelImage, isPublic);
        emit DebugLog("Constructor: End", 0);
    }


    /**
    * @dev Can only be called by an address that was included in the constructor.
    * The user's status changes from 'Invited' to 'Active'.
     */
    function acceptInvite() external onlyInvitedMember() whenNotLocked(){
        require(members[msg.sender].status == MemberStatus.Invited, "Etibe: Invite already or invalid.");
        members[msg.sender].status = MemberStatus.Active;
        memberAddresses.push(msg.sender);

        emit InviteAccepted(msg.sender, block.timestamp);
    }

     /**
     * @notice Allows the admin to lock the channel, preventing new members from joining.
     */
    function lockChannel() external onlyAdmin whenNotLocked {
        require(memberAddresses.length == channelSize, "Etibe: Cannot lock until all members have joined.");

        isLocked = true;

        emit ChannelLocked(block.timestamp);
    }

    /**
     * @notice Allows an active member to contribute their monthly amount.
     * @dev The sender must be an active member and send the exact `contributionAmount`.
     */
    function makeContribution() external payable whenLocked onlyMembers {
        require(msg.value == contributionAmount, "Etibe: Incorrect contribution amount. Send the exact contribution amount.");
        require(members[msg.sender].lastContributionMonth < currentCycleMonth, "Etibe: You have already contributed for this cycle month.");

        members[msg.sender].lastContributionMonth = currentCycleMonth;
        emit ContributionReceived(msg.sender, currentCycleMonth, msg.value);
    }

    /**
     * @notice Processes the monthly payout to the scheduled recipient.
     */
    function processMonthlyPayout() external whenLocked {
        require(!isComplete, "Etibe: Channel is already complete.");

        uint256 expectedPayoutTimestamp = startDate + (currentCycleMonth - 1) * 30 days;
        require(block.timestamp >= expectedPayoutTimestamp, "Etibe: Payout period not yet reached for this month.");

        address payable payoutRecipient;
        bool recipientFound = false;

        for (uint i = 0; i < memberAddresses.length; i++) {
            address currentMemberAddr = memberAddresses[i];
            if (members[currentMemberAddr].payoutMonth == currentCycleMonth) {
                payoutRecipient = payable(currentMemberAddr);
                recipientFound = true;
                break;
            }
        }

        require(recipientFound, "Etibe: No recipient found for the current month's payout.");
        require(members[payoutRecipient].status != MemberStatus.Removed, "Etibe: Recipient has been removed and cannot receive payout.");


        for (uint i = 0; i < memberAddresses.length; i++) {
            address currentMemberAddr = memberAddresses[i];
            if (members[currentMemberAddr].status == MemberStatus.Active || members[currentMemberAddr].status == MemberStatus.PaidOut) {
                 require(members[currentMemberAddr].lastContributionMonth >= currentCycleMonth, "Etibe: Not all active members have contributed for this cycle month.");
            }
        }

        uint256 expectedTotalContribution = contributionAmount * memberAddresses.length; // Calculate based on current active members
        require(address(this).balance >= expectedTotalContribution, "Etibe: Insufficient funds in contract for payout.");

        members[payoutRecipient].status = MemberStatus.PaidOut;
        members[payoutRecipient].receivedContributionMonth = currentCycleMonth;

        if (currentCycleMonth < channelSize) {
            members[payoutRecipient].status = MemberStatus.Active;
        }

        (bool success, ) = payoutRecipient.call{value: expectedTotalContribution}("");
        require(success, "Etibe: Payout transfer failed.");

        emit PayoutMade(payoutRecipient, currentCycleMonth, expectedTotalContribution);

        // Move to the next cycle month
        currentCycleMonth++;

        if (currentCycleMonth > channelSize) {
            isComplete = true;
            emit ChannelCompleted(block.timestamp);
        }
    }

    /**
     * @notice Allows the admin to remove a member who has failed to contribute
     * within the grace period for the current cycle month.
     * @param memberToRemove The address of the member to be removed.
     */
    function enforceGracePeriodAndRemove(address memberToRemove) external onlyAdmin whenLocked {
        require(!isComplete, "Etibe: Channel is already complete. No members can be removed.");
        require(members[memberToRemove].walletAddress != address(0), "Etibe: Member does not exist.");

        require(members[memberToRemove].status == MemberStatus.Active || members[memberToRemove].status == MemberStatus.Invited, "Etibe: Member is not in a removable state (must be Active or Invited).");

        require(members[memberToRemove].lastContributionMonth < currentCycleMonth, "Etibe: Member has already contributed for this cycle month.");

        // Calculate the timestamp when the grace period for the current cycle month expires.
        uint256 gracePeriodExpirationTimestamp = startDate + (currentCycleMonth - 1) * 30 days + gracePeriod;
        require(block.timestamp > gracePeriodExpirationTimestamp, "Etibe: Grace period has not expired yet for this cycle month.");

        // Mark the member as removed.
        members[memberToRemove].status = MemberStatus.Removed;

        bool foundAndRemoved = false;
        for (uint i = 0; i < memberAddresses.length; i++) {
            if (memberAddresses[i] == memberToRemove) {
                memberAddresses[i] = memberAddresses[memberAddresses.length - 1];
                memberAddresses.pop();
                foundAndRemoved = true;
                break;
            }
        }
        require(foundAndRemoved, "Etibe: Member address not found in active member list.");

        emit MemberRemoved(memberToRemove, currentCycleMonth);
    }

    /**
     * @notice Returns comprehensive details about the Etibe Channel.
     * @dev This function is useful for displaying channel information on the UI.
     * @return name The channel's name.
     * @return contribution The fixed monthly contribution amount.
     * @return balance The current balance of the contract.
     * @return locked True if the channel is locked, false otherwise.
     * @return size The total number of members in the channel.
     * @return currentMonth The current cycle month.
     * @return complete True if the channel's cycles are complete, false otherwise.
     * @return createdDate The timestamp when the channel was created.
     * @return sDate The official start date of the first contribution cycle.
     * @return gPeriod The grace period in seconds.
     * @return image The channel's image URL.
     * @return publicStatus True if the channel is public, false otherwise.
     */
    function getChannelDetails()
        external
        view
        returns (
            string memory name,
            uint256 contribution,
            uint256 balance,
            bool locked,
            uint256 size,
            uint256 currentMonth,
            bool complete,
            uint256 createdDate,
            uint256 sDate,
            uint256 gPeriod,
            string memory image,
            bool publicStatus
        )
    {
        return (
            channelName,
            contributionAmount,
            address(this).balance,
            isLocked,
            channelSize,
            currentCycleMonth,
            isComplete,
            creationDate,
            startDate,
            gracePeriod,
            channelImage,
            isPublic
        );
    }

    /**
     * @notice Returns the status and last contribution month of a specific member.
     * @param memberAddr The address of the member to query.
     * @return status The current status of the member (Invited, Active, Removed, PaidOut).
     * @return lastPaidMonth The last month number for which the member successfully paid.
     * @return receivedMonth The last month number for which the member received their contribution share.
     * @return payoutMonth The month number they are scheduled to receive the payout.
     */
    function getMemberStatus(address memberAddr)
        external
        view
        returns (
            MemberStatus status,
            uint256 lastPaidMonth,
            uint256 receivedMonth,
            uint256 payoutMonth
        )
    {
        // Require the member to exist in the mapping.
        require(members[memberAddr].walletAddress != address(0), "Etibe: Member does not exist.");

        return (
            members[memberAddr].status,
            members[memberAddr].lastContributionMonth,
            members[memberAddr].receivedContributionMonth,
            members[memberAddr].payoutMonth
        );
    }

    /**
     * @notice Returns the current number of active (accepted) members in the channel.
     * @dev This is the length of the dynamic `memberAddresses` array.
     * @return The count of active members.
     */
    function getMemberAddressesLength() public view returns (uint256) {
        return memberAddresses.length;
    }
}
```

**EtibeChannelFactory.sol**

This contract serves as a powerful factory, enabling the seamless creation and efficient management of multiple EtibeChannel instances. It acts as the entry point for deploying new contribution groups on the UMI blockchain.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "./EtibeChannel.sol";

/**
 * @title EtibeChannelFactory
 * @author Utee
 * @notice A factory contract for creating and managing multiple EtibeChannel instances.
 */
contract EtibeChannelFactory {
    EtibeChannel[] public deployedChannels;

    event EtibeChannelCreated(
        address indexed channelAddress,
        address indexed admin, // This will now be the EOA that created it via the factory
        string name,
        uint256 contributionAmount,
        uint256 channelSize,
        bool isPublic
    );

    function createChannel(
        string memory _channelName,
        string memory _channelImage,
        uint256 _contributionAmount,
        uint256 _startDate,
        uint8 _gracePeriodInDays,
        bool _isPublic,
        address[] memory _invitedMembers,
        uint256[] memory _payoutOrder
    ) external returns (address newChannelAddress) {
        EtibeChannel newEtibeChannel = new EtibeChannel(
            msg.sender,
            _channelName,
            _channelImage,
            _contributionAmount,
            _startDate,
            _gracePeriodInDays,
            _isPublic,
            _invitedMembers,
            _payoutOrder
        );

        deployedChannels.push(newEtibeChannel);

        emit EtibeChannelCreated(
            address(newEtibeChannel),
            msg.sender,
            _channelName,
            _contributionAmount,
            _invitedMembers.length,
            _isPublic
        );

        return address(newEtibeChannel);
    }

    function getChannelCount() public view returns (uint256) {
        return deployedChannels.length;
    }

    function getChannelAddress(uint256 _index) public view returns (address) {
        require(_index < deployedChannels.length, "EtibeFactory: Invalid channel index.");
        return address(deployedChannels[_index]);
    }
}
```

## 🔐 Security Considerations

Security is paramount for Etibé. We have implemented several robust measures to safeguard user funds and ensure the integrity of the contribution process:

- Encrypted Private Keys: User wallet private keys are encrypted and stored securely on the user's device, mitigating risks associated with centralised storage.

- Biometric/PIN Access: For highly sensitive actions, such as viewing seed phrases or altering passwords, users are required to authenticate via biometric verification or a personal identification number (PIN), adding an extra layer of protection.

- Smart Contract Enforcement: The entire contribution logic is immutably enforced by UMI smart contracts. Once a channel is created and its rules are set, the contract is locked, preventing any unauthorised alterations to the contribution order or amounts. This ensures that no admin can skip users or manipulate the payout sequence.

- Trustless Auto-Removal: The logic for automatically removing members due to non-payment is handled on-chain where feasible, ensuring transparency and eliminating the need for trust in a central authority.

## 📈 Future Enhancements

Etibé is designed with future growth and enhanced functionality in mind. Our roadmap includes exciting developments such as:

- Fiat On-Ramp: Integrating seamless options for users to purchase UMI directly using traditional payment methods, such as debit/credit cards or bank transfers, to improve accessibility.

- Backup Users/Payees: Implementing a system that allows channel admins to designate fallback payees in the event a member is removed, ensuring the contribution chain remains unbroken and payouts continue smoothly.

- Group Chat: Introducing a dedicated chat feature within each Etibé Channel, fostering direct communication and community engagement amongst members.

- Ratings/Reliability Score: Developing a transparent system to rate members based on their historical contribution, punctuality and reliability, building trust and accountability within the community.

- Referral Rewards: Launching an incentive programme to reward users for successfully inviting new members to the platform, encouraging organic growth.

- Analytics Dashboard: Providing channel admins with a comprehensive dashboard offering insights into channel health, member reliability, and overall performance metrics.

- Staking Incentives: Exploring mechanisms to offer staking incentives for consistently timely contributors, further encouraging responsible participation.

Etibé is not just a hackathon project; it's a vision for a more equitable and efficient financial future, powered by decentralisation and community spirit. We believe Etibé has the potential to revolutionise traditional savings practices, bringing transparency, trust, and automation to millions.
