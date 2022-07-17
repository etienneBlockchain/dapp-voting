<script setup>
	import { RouterLink, RouterView } from 'vue-router'
	import { BootstrapVue, BootstrapVueIcons } from 'bootstrap-vue'
	import VotingContract from '@/contracts/Voting.json'
	import getWeb3 from '@/utils/getWeb3'

	import 'bootstrap/dist/css/bootstrap.css'
	import 'bootstrap-vue/dist/bootstrap-vue.css'
	import 'bootstrap-icons/font/bootstrap-icons.css'
	import '@/assets/base.css'
</script>

<template>
	<header class="navbar navbar-dark mb-5">
		<nav class="container">
			<p class="navbar-brand align-middle mb-0">Alyra - Voting Project - Etienne H.</p>
			<div class="text-right text-light">
				<p class="mb-0">
					<span v-if="!connectedWallet" class="badge badge-pill badge-danger">Wallet not connected</span>
					<span v-if="connectedWallet" class="badge badge-pill badge-success">Wallet connected</span>
				</p>
				<p v-if="connectedWallet" class="mb-0 small">
					{{ connectedWallet }}
					<span v-if="currentOwner">(Owner)</span>
				</p>
			</div>
		</nav>
	</header>

	<div class="container mb-3">
		<div v-if="!connectedWallet" class="alert alert-danger" role="alert">Please connect your wallet to access the application.</div>
		<div v-else>
			<div v-if="successAlert.show" class="alert alert-success mb-5" role="alert">{{ successAlert.message }}</div>

			<div class="row">
				<div class="col-8">
					<div v-if="currentOwner && workflowStatus == 0" class="mb-5">
						<label for="add-address" class="font-weight-bold">Add a voter</label>
						<div class="form-inline">
							<input type="text" autocomplete="off" class="form-control col-6" v-model="addVoter" placeholder="Address" id="add-address">
							<button class="btn btn-perso ml-3" @click="addVoterAddress()"><i class="bi bi-person-plus mr-2"></i> Add a voter</button>
						</div>
						<div v-if="errorAddVoter" class="text-danger mt-1 small">{{ errorAddVoter }}</div>
					</div>

					<div v-if="currentVoter" class="mb-5">
						<label for="search-address" class="font-weight-bold">Search a voter</label>
						<div class="form-inline">
							<input autocomplete="off" type="text" class="form-control col-6" v-model="searchVoter" placeholder="Address" id="search-address">
							<button class="btn btn-perso ml-3" @click="searchVoterAddress()"><i class="bi bi-search mr-2"></i> Search</button>
						</div>
						<div v-if="errorSearchVoter" class="text-danger mt-1 small">{{ errorSearchVoter }}</div>
		
						<ul class="list-group mt-4" v-if="searchResult">
							<li class="list-group-item">
								<strong>Address:</strong> {{ searchResult.address }}
							</li>
							<li class="list-group-item">
								<strong>Is registered:</strong>
								<span v-if="searchResult.registered" class="ml-2"><i class="bi bi-check-circle-fill text-success"></i></span>
								<span v-else class="ml-2"><i class="bi bi-x-circle-fill text-danger"></i></span>
							</li>
							<li class="list-group-item">
								<strong>Has voted:</strong>
								<span v-if="searchResult.voted" class="ml-2"><i class="bi bi-check-circle-fill text-success"></i></span>
								<span v-else class="ml-2"><i class="bi bi-x-circle-fill text-danger"></i></span>
							</li>
						</ul>
					</div>

					<div v-if="currentVoter && workflowStatus == 1" class="mb-5">
						<label for="description-proposal" class="font-weight-bold">Add a proposal</label>
						<div class="form-inline">
							<input autocomplete="off" type="text" class="form-control col-6" v-model="addProposal" placeholder="Description" id="description-proposal">
							<button class="btn btn-perso ml-3" @click="addProposalDescription()"><i class="bi bi-plus-circle mr-2"></i> Add a proposal</button>
						</div>
						<div v-if="errorAddProposal" class="text-danger mt-1 small">{{ errorAddProposal }}</div>

						<ul class="list-group mt-5">
							<li class="list-group-item text-light font-weight-bold header-list">Proposals list</li>
							<li class="list-group-item" v-show="!proposalsList[0]">No proposal</li>
							<li v-for="proposal in proposalsList" class="list-group-item">
								<strong>{{ proposal.id }}</strong> - {{ proposal.label }}
							</li>
						</ul>
					</div>

					<div v-if="currentVoter && workflowStatus == 3">
						<ul class="list-group mt-5">
							<li class="list-group-item text-light font-weight-bold header-list">Proposals list</li>
							<li v-for="proposal in proposalsList" class="list-group-item align-middle">
								<div class="row">
									<div class="col-8 align-self-center">
										<strong>{{ proposal.id }}</strong> - {{ proposal.label }}
									</div>
									<div class="col-4 text-right align-self-center">
										<button class="btn btn-perso" @click="vote(proposal.id)">
											<i class="bi bi-envelope-paper mr-2"></i> Vote
										</button>
									</div>
								</div>
							</li>
						</ul>
					</div>

					<div v-if="workflowStatus == 5 && winner"> 
						<h2>Winner</h2>
						<div class="alert alert-success mt-3 text-center" role="alert">
							<p class="mb-0">The winning proposal is <strong>{{ winner.label }}</strong></p>
						</div>
					</div>
				</div>
				<div class="col-4">
					<h5>States</h5>
					<ul class="list-group">
						<li class="list-group-item" v-for="(name, key) in workflowStatusName" :class="(key == workflowStatus) ? 'list-group-item-success' : 'disabled'">
							<strong>{{ key + 1 }}</strong> - {{ name }}
							<strong v-if="key == workflowStatus" class="small font-weight-bold">(current)</strong>
						</li>
					</ul>
					<p class="mt-4 text-center" v-if="currentOwner">
						<button class="btn btn-perso" @click="changeStatus()" v-if="workflowStatus < 5">
							{{ workflowStatusName[(workflowStatus * 1 ) + 1] }}
							<i class="bi bi-arrow-right-short"></i>
						</button>
					</p>
				</div>
			</div>
		</div>
	</div>
</template>

<script>
	export default {
		data() {
			return {
				web3: false,
				accounts: false,
				connectedWallet: false,
				owner: false,
				instance: false,
				currentVoter: false,
				currentOwner: false,
				addVoter: null,
				addProposal: null,
				proposalsList: false,
				winner: false,
				successAlert: {
					show: false,
					message: false
				},
				errorAddVoter: false,
				errorAddProposal: false,
				searchVoter: null,
				errorSearchVoter: false,
				workflowStatus: false,
				searchResult: false,
				workflowStatusName: [
					'Registering Voters',
					'Proposals Registration Started',
					'Proposals Registration Ended',
					'Voting Session Started',
					'Voting Session Ended',
					'Votes Tallied'
				]
			}
		},
		methods: {
			async checkConnectedWallet () {
				this.web3 = await getWeb3()
				this.accounts = await this.web3.eth.getAccounts()
				this.connectedWallet = this.accounts[0]
				
				const networkId = await this.web3.eth.net.getId()
				const deployedNetwork = VotingContract.networks[networkId]
				this.instance = await new this.web3.eth.Contract(VotingContract.abi, deployedNetwork && deployedNetwork.address)
				this.owner = await this.instance.methods.owner().call()

				this.checkCurrentIsOwner()
				this.checkCurrentIsVoter()
			},
			async vote (idProposal) {
				this.successAlert.show = false
				try {
					await this.instance.methods.setVote(idProposal).send({ from: this.accounts[0] })
					this.successAlert = {
						show: true,
						message: 'Your vote has been recorded.'
					}
				} catch (error) {
					this.instance.methods.setVote(idProposal).call({ from: this.accounts[0] })
					.then(result => { throw Error('unlikely to happen') })
					.catch(revert => {
						revert = revert.message.split('Internal JSON-RPC error.')[1].trim()
						revert = JSON.parse(revert)
						alert(revert.message)
					})
				}
			},
			async getProposalsEvent () {
				const proposalsList = await this.instance.getPastEvents('ProposalRegistered', { fromBlock: 0 })

				let proposals = []
				for (let i = 0; i < proposalsList.length; i++) {
					const label = await this.instance.methods.getOneProposal(i).call({ from: this.accounts[0] })
					proposals[i] = {
						label: label.description,
						id: i
					}
				}

				this.proposalsList = proposals
			},
			checkCurrentIsOwner () {
				this.currentOwner = false
				if (this.owner == this.connectedWallet) {
					this.currentOwner = true
				}
			},
			async changeStatus () {
				this.successAlert.show = false
				this.searchResult = false

				if (this.workflowStatus == 0) {
					await this.instance.methods.startProposalsRegistering().send({ from: this.accounts[0] })
					this.getWorkflowStatus()
				} else if (this.workflowStatus == 1) {
					await this.instance.methods.endProposalsRegistering().send({ from: this.accounts[0] })
					this.getWorkflowStatus()
				} else if (this.workflowStatus == 2) {
					await this.instance.methods.startVotingSession().send({ from: this.accounts[0] })
					this.getWorkflowStatus()
				} else if (this.workflowStatus == 3) {
					await this.instance.methods.endVotingSession().send({ from: this.accounts[0] })
					this.getWorkflowStatus()
				} else if (this.workflowStatus == 4) {
					this.workflowStatus = 5
					await this.tallyVotes()
				}
			},
			async tallyVotes () {
				try {
					await this.instance.methods.tallyVotes().send({ from: this.accounts[0] })
					const winner = await this.instance.methods.winningProposalID().call()
					const label = await this.instance.methods.getOneProposal(winner).call({ from: this.accounts[0] })
					this.winner = {
						id: winner,
						label: label.description
					}
				} catch (error) {
					this.instance.methods.tallyVotes().call({ from: this.accounts[0] })
					.then(result => { throw Error('unlikely to happen') })
					.catch(revert => {
						revert = revert.message.split('Internal JSON-RPC error.')[1].trim()
						revert = JSON.parse(revert)
						alert(revert.message)
					})
				}
			},
			async checkCurrentIsVoter () {
				this.currentVoter = false
				const data = await this.instance.methods.getVoter(this.connectedWallet).call({ from: this.accounts[0] })
				if (data.isRegistered) {
					this.currentVoter = true
				}
			},
			async getWorkflowStatus () {
				this.workflowStatus = await this.instance.methods.workflowStatus().call()
			},
			async addProposalDescription () {
				this.errorAddProposal = false
				this.successAlert.show = false
				if (this.addProposal) {
					this.addProposal = this.addProposal.trim()
					try {
						await this.instance.methods.addProposal(this.addProposal).send({ from: this.accounts[0] })
						this.addProposal = null
						this.successAlert = {
							show: true,
							message: 'The new proposal has been registered'
						}
						this.getProposalsEvent()
					} catch (error) {
						this.instance.methods.addProposal(this.addProposal).call({ from: this.accounts[0] })
						.then(result => { throw Error('unlikely to happen') })
						.catch(revert => {
							revert = revert.message.split('Internal JSON-RPC error.')[1].trim()
							revert = JSON.parse(revert)
							this.errorAddProposal = revert.message
						})
					}
				} else {
					this.errorAddProposal = 'Please enter a description.'
				}
			},
			async addVoterAddress () {
				this.errorAddVoter = false
				this.successAlert.show = false
				if (this.addVoter) {
					this.addVoter = this.addVoter.trim()
					if (this.web3.utils.isAddress(this.addVoter)) {
						try {
							await this.instance.methods.addVoter(this.addVoter).send({ from: this.accounts[0] })
							this.addVoter = null
							this.successAlert = {
								show: true,
								message: 'The new voter has been registered'
							}

							this.checkCurrentIsVoter()

						} catch (error) {
							this.instance.methods.addVoter(this.addVoter).call({ from: this.accounts[0] })
							.then(result => { throw Error('unlikely to happen') })
							.catch(revert => {
								revert = revert.message.split('Internal JSON-RPC error.')[1].trim()
								revert = JSON.parse(revert)
								this.errorAddVoter = revert.message
							})
						}
					} else {
						this.errorAddVoter = 'The address is invalid'
					}
				} else {
					this.errorAddVoter = 'Please enter an address'
				}
			},
			async searchVoterAddress () {
				this.errorSearchVoter = false
				if (this.searchVoter) {
					this.searchVoter = this.searchVoter.trim()
					if (this.web3.utils.isAddress(this.searchVoter)) {
						try {
							const data = await this.instance.methods.getVoter(this.searchVoter).call({ from: this.accounts[0] })
							this.searchResult = {
								address: this.searchVoter,
								registered: data.isRegistered,
								voted: data.hasVoted
							}
							this.searchVoter = null
						} catch (error) {
							console.log(error)
						}
					} else {
						this.errorSearchVoter = 'The address is invalid'
					}
				} else {
					this.errorSearchVoter = 'Please enter an address'
				}
			}
		},
		async mounted () {
			await this.checkConnectedWallet()
			await this.getWorkflowStatus()
			
			await this.getProposalsEvent()
			
			// Reload connected Wallet is user change our wallet in Metamask
			ethereum.on('accountsChanged', accounts => {
				this.successAlert.show = false
				this.checkConnectedWallet()
			})
		}
	}
</script>
