<template>
	<div v-if="identity !== null">
	<h1>
		<i v-if="identity.privateKey !== null" class="fa fa-key"></i> 
		<i v-else class="fa fa-user"></i>
		{{$t('identity')}} <catena-hash :hash="identity.publicHash" format="base64"></catena-hash></h1>
	<dl>
		<template v-if="identity !== null">
			<dt>{{$t('publicKey')}}</dt>
			<dd><code>{{identity.publicBase58}}</code></dd>
		</template>

		<template v-if="identity !== null">
			<dt>{{$t('publicHashHex')}}</dt>
			<dd><code>X'{{identity.publicHashHex}}'</code></dd>
		</template>

		<template v-if="identity !== null">
			<dt>{{$t('counter')}}</dt>
			<dd>{{counter}} <button @click="updateCounter">{{$t('refresh')}}</button></dd>
		</template>

		<template v-if="identity !== null && identity.privateKey !== null">
			<dt>{{$t('privateKey')}}</dt>
			<dd>
				<catena-expander :title="$t('showPrivateKey')" icon="key">
					<code>{{identity.privateBase58}}</code>
				</catena-expander>
			</dd>
		</template>
	</dl>

	<h2>{{$t('ownedDatabases')}}</h2>
	<catena-query v-if="agent !== null" :sql="ownedSQL" :agent="agent" :head="head" :database="database" :no-data="$t('noOwnedDatabases')"></catena-query>

	<h2>{{$t('grants')}}</h2>
	<dl>
		<dt>{{$t('showGrantsInDatabase')}}</dt>
		<dd>
			<select v-model="database">
				<option key="" value="">{{$t('selectDatabase')}}</option>
				<option v-for="db in databases" :key="db" :value="db">{{db}}</option>
			</select>
		</dd>
	</dl>

	<catena-query v-if="agent !== null && database != '' " :sql="grantsSQL" :agent="agent" :head="head" :database="database" :no-data="$t('noPrivileges')"></catena-query>
	<catena-expander :title="$t('grantPrivileges')" icon="plus" v-if="database != '' ">
		<catena-granter :agent="agent" :user="identity" :database="database"></catena-granter>
	</catena-expander>

	<template v-if="identity !== null">
		<h2>{{$t('messaging')}}</h2>
		<catena-expander title="Sign a message" icon="hand-spock-o" v-if="identity.privateKey !== null">
			<textarea class="catena-code" v-model="messageToSign" @keyup="clearSignature"></textarea>
			<button @click="signMessage">{{$t('sign')}}</button> <button @click="clearSignMessage">{{$t('clearMessage')}}</button>

			<dl v-if="messageSignature !== null">
				<dt>{{$t('signature')}}</dt>
				<dd><input type="text" style="width: 100%;" readonly disabled :value="messageSignature"/></dd>

				<dt>{{$t('combined')}}</dt>
				<dd><input type="text" style="width: 100%;" readonly disabled :value="combinedSignedMessage"/></dd>
			</dl>
		</catena-expander>

		<catena-expander title="Verify a message" icon="handshake-o">
			<textarea class="catena-code" v-model="messageToVerify" @keyup="clearVerify" placeholder="Message (may be combined)"></textarea>
			<input type="text" style="width: 100%;" :placeholder="'Public key (leave empty to use '+identity.publicBase58+')' " v-model="verifyPublicKey" @keyup="clearVerify"/>
			<input type="text" style="width: 100%;" placeholder="Signature" v-model="verifySignature" @keyup="clearVerify"/>
			<button @click="verifyMessage">{{$t('verify')}}</button> <button @click="clearVerifyMessage">{{$t('clearMessage')}}</button>

			<template v-if="messageVerified !== null">
				<p class="info" v-if="messageVerified">
					<i class="fa fa-check"></i> {{$t('messageVerified')}}
				</p>
				<p class="error" v-else>
					<i class="fa fa-times"></i> {{$t('messageDidNotVerify')}}
				</p>
			</template>
		</catena-expander>
	</template>

	<h2>{{$t('storage')}}</h2>
	<p v-if="!isPersisted">{{$t('storeDescription')}}</p>
	<p v-if="isPersisted">{{$t('storedDescription')}}</p>
	<button @click="persist" v-if="!isPersisted"><i class="fa fa-bookmark-o"></i>{{$t('persist')}}</button>
	<button @click="forget" v-if="isPersisted"><i class="fa fa-eraser"></i> {{$t('forget')}}</button>
	</div>
</template>

<script>
const base64 = require('base64-js');
const Identity = require("./blockchain").Identity;
const Agent = require("./blockchain").Agent;
const Transaction = require("./blockchain").Transaction;

module.exports = {
	props: {
		identity: Identity,
		agent: Agent,
		head: {type: String, default: null}
	},

	data: function() {
		return { 
			database: "",
			databases: [],
			isPersisted: this.identity ? this.identity.isPersisted : false,
			counter: null,
			messageToSign: "",
			messageSignature: null,
			messageToVerify: null,
			verifySignature: null,
			messageVerified: null,
			verifyPublicKey: null,
		};
	},

	i18n: { messages: {
		en: {
			privateKey: "Secret key",
			publicKey: "Public key",
			showPrivateKey: "Show",
			counter: "Counter",
			publicHashHex: "Public key hash (hex)",
			refresh: "Refresh",
			storage: "Storage",
			storeDescription: "Save this identity in this browser's local storage.",
			storedDescription: "This identity is saved in the local storage of this browser.",
			persist: "Persist",
			forget: "Forget",
			messageDidNotVerify: "The message did not verify",
			messageVerified: "The message verified successfully",
			clearMessage: "Clear",
			messaging: "Messaging",
			signature: "Signature",
			combined: "Combined",
			noPrivileges: "This user was not granted any privileges for this database.",
			grants: "Grants",
			ownedDatabases: "Owned databases",
			showGrantsInDatabase: "Show grants in database:",
			selectDatabase: "Select a database...",
			sign: "Sign",
			verify: "Verify",
			noOwnedDatabases: "This user does not currently own any databases.",
			grantPrivileges: "Grant privileges",
			identity: "Identity",
		},

		nl: {
			privateKey: "Geheime sleutel",
			publicKey: "Publieke sleutel",
			showPrivateKey: "Toon",
			counter: "Teller",
			publicHashHex: "Hash van publieke sleutel (hex)",
			refresh: "Vernieuw",
			storage: "Opslag",
			storeDescription: "Sla deze identiteit in de browser op deze computer op.",
			storedDescription: "Deze identiteit is opgeslagen in de browser op deze computer.",
			persist: "Onthoud",
			forget: "Vergeet",
			messageDidNotVerify: "Het bericht kon niet worden geverifieerd",
			messageVerified: "Het bericht is met succes geverifieerd",
			clearMessage: "Wis",
			messaging: "Berichtuitwisseling",
			signature: "Handtekening",
			combined: "Gecombineerd",
			noPrivileges: "Aan deze gebruiker zijn geen privileges verleend in deze database.",
			grants: "Verleende privileges",
			ownedDatabases: "Databases waarvan deze gebruiker eigenaar is",
			showGrantsInDatabase: "Toon aan deze gebruiker verleende privileges in database:",
			selectDatabase: "Selecteer een database...",
			sign: "Onderteken",
			verify: "Verifieer",
			noOwnedDatabases: "Er zijn geen databases waarvan deze gebruiker eigenaar is.",
			grantPrivileges: "Verleen privileges",
			identity: "Identiteit",
		}
	} },

	watch: {
		identity: function(nv) {
			this.updateCounter();
		},

		head: function() {
			this.refreshDatabases();
		}
	},

	created: function() {
		this.updateCounter();
		this.refreshDatabases();
	},

	computed: {
		grantsSQL: function() {
			return "SHOW GRANTS FOR X'"+this.identity.publicHashHex+"';";
		},

		ownedSQL: function() {
			return "SHOW DATABASES FOR X'"+this.identity.publicHashHex+"';";
		},

		combinedSignedMessage: function() {
			return JSON.stringify([this.messageToSign, this.messageSignature]);
		}
	},

	methods: {
		refreshDatabases: function() {
			var self = this;

			this.agent.databases(function(err, databases) {
				databases.sort();
				self.databases = databases;
			});
		},

		verifyMessage: function() {
			if(this.verifySignature === null || this.verifySignature.length == 0) {
				try {
					if(this.messageToVerify === null || this.messageToVerify.length == 0) {
						alert("Please enter a message to verify");
					}
					else {
						let m = JSON.parse(this.messageToVerify);
						if(Array.isArray(m) && m.length == 2) {
							this.messageToVerify = m[0];
							this.verifySignature = m[1];
						}
					}
				}
				catch(e) {
					alert("Please enter a signature");
					return;
				}
			}

			try {
				let sig = base64.toByteArray(this.verifySignature);

				if(this.verifyPublicKey === null) {
					this.messageVerified = this.identity.verify(new Buffer(this.messageToVerify), new Buffer(sig));
				}
				else {
					this.messageVerified = Identity.verify(new Buffer(this.messageToVerify), this.verifyPublicKey, new Buffer(sig));
				}
			}
			catch(e) {
				this.messageVerified = false;
				alert(e);
			}
		},

		clearSignMessage: function() {
			this.messageToSign = null;
			this.messageSignature = null;
		},

		clearVerifyMessage: function() {
			this.messageVerified = null;
			this.messageToVerify = null;
			this.verifySignature = null;
			this.verifyPublicKey = null;
		},

		clearVerify: function() {
			this.messageVerified = null;
		},

		clearSignature: function() {
			this.messageSignature = null;
		},

		updateCounter: function() {
			var self = this;
			this.counter = null;

			this.agent.counter(this.identity.publicBase58, function(err, ctr) {
				self.counter = ctr || 0;
			});
		},

		signMessage: function() {
			let buffer = new Buffer(this.messageToSign);
			this.messageSignature = base64.fromByteArray(this.identity.sign(buffer));
		},

		persist: function() {
			this.identity.persist(true);
			this.isPersisted = this.identity.isPersisted;
		},

		forget: function() {
			this.identity.persist(false);
			this.isPersisted = this.identity.isPersisted;
		}
	}
};
</script>