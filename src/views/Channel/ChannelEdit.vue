<template>
    <div class="text-center space-y-2">
        <h3 v-if="channel[channelKey]">{{ channel[channelKey].channelName }}</h3>
        <div>
            <Button :pressed="this.isSubscribed(channelKey)" @click="handleSubscribe(channelKey)">Subscribe{{ this.isSubscribed(channelKey) ? 'd' : '' }}</Button>
        </div>
        <div class="space-y-2 pt-4">
            <span class="font-medium" v-if="upcomingTasks.length > 0">Upcoming tasks:</span>
            <div class="flex flex-col items-center" v-for="task in upcomingTasks" :key="task['.key']">
                <button class="w-5/6 lg:w-3/6 selectable" @click="gotoTask(task['.key'])">
                    <Card class="item">
                        <div class="font-bold font-display truncate">{{ task.name }}</div>
                        <div class="text-sm flex items-center space-x-2"><CalendarIcon size="1x"/> <div>{{ task.deadlineDate | dateFormat }}</div></div>
                    </Card>
                </button>
            </div>
        </div>

        <div class="space-y-2 pt-4">
            <span class="font-medium" v-if="pastTasks.length > 0">Past tasks:</span>
            <div class="flex flex-col items-center" v-for="task in pastTasks" :key="task['.key']">
                <button class="w-5/6 lg:w-3/6 selectable" @click="gotoTask(task['.key'])">
                    <Card class="item">
                        <div class="font-bold font-display truncate">{{ task.name }}</div>
                        <div class="text-sm flex items-center space-x-2"><CalendarIcon size="1x"/> <div>{{ task.deadlineDate | dateFormat }}</div></div>
                    </Card>
                </button>
            </div>
        </div>

        <floating-action-button primary @click="addNewTask()" v-if="user"><PlusIcon/></floating-action-button>
    </div>
</template>

<script>
    import Card from "../../components/Card";
    import moment from "moment"
    import { CalendarIcon, PlusIcon } from 'vue-feather-icons'
    import FloatingActionButton from "../../components/FloatingActionButton";
    import Button from "../../components/Button";
    export default {
        name: "ChannelEdit",
        components: {Button, FloatingActionButton, Card, CalendarIcon, PlusIcon},
        data() {
            return {
                channel: {},
                tasks: [],
                channelId: 0,
            }
        },
        computed: {
            channelKey() {
                return Object.keys(this.channel)[0]
            },
            upcomingTasks() {
                let upcomingTasks = []
                this.tasks.forEach(task => {
                    if (moment().diff(task['deadlineDate'], 'days') <= 0) upcomingTasks.push(task)
                })
                return upcomingTasks
            },
            pastTasks() {
                let pastTasks = []
                this.tasks.forEach(task => {
                    if (moment().diff(task['deadlineDate'], 'days') > 0) pastTasks.push(task)
                })
                return pastTasks
            },
            user() {
                return this.$store.state.user
            },
            userData() {
                return this.$store.state.userData
            },
            subscriptions() {
                return this.$store.state.subscriptions
            }
        },
        mounted() {
            if (!parseInt(this.$route.params.id)) {
                this.$router.push('/')
            } else {
                this.channelId = parseInt(this.$route.params.id)

                // get current user
                this.$store.dispatch('getAuthenticatedUser')

                // bind user data
                this.$store.dispatch('bindSubscriptions')
            }
        },
        methods: {
            bindChannel(channelId) {
                const dbQuery = this.$firebase.database().ref('channels').orderByChild('id').equalTo(channelId).limitToFirst(1)

                this.$rtdbBind('channel', dbQuery).catch(e => {
                    console.log(e.message)
                })
            },
            bindTasks(channelKey) {
                const dbQuery = this.$firebase.database().ref('tasks').orderByChild('channelKey').equalTo(channelKey)

                this.$rtdbBind('tasks', dbQuery).catch(e => {
                    console.log(e.message)
                })
            },
            addNewTask() {
                this.$router.push(`/dashboard/channels/${this.channelId}/task/new`)
            },
            gotoTask(taskKey) {
                this.$router.push(`${this.$route.path}/task/${taskKey}`)
            },
            isSubscribed(channelKey) {
                if (this.subscriptions) {
                    const subscriptionKeys = Object.keys(this.subscriptions)
                    for (let x=0; x<subscriptionKeys.length; ++x) {
                        const subscriptionKey = subscriptionKeys[x]
                        const subscription = this.subscriptions[subscriptionKey]

                        if (subscription.channelKey === channelKey) {
                            return true
                        }
                    }
                }
                return false
            },
            handleSubscribe(channelKey) {
                if (this.isSubscribed(channelKey)) this.doUnsubscribe(channelKey)
                else this.doSubscribe(channelKey)
            },
            doSubscribe(channelKey) {
                const dbRef = this.$firebase.database().ref(`subscriptions/`)
                const pushObj = {
                    channelKey,
                    timestamp: Date.now(),
                    userId: this.user.uid
                }
                dbRef.push(pushObj)
            },
            doUnsubscribe(channelKey) {
                const subscriptionKeys = Object.keys(this.subscriptions)
                for (let x=0; x<subscriptionKeys.length; ++x) {
                    const subscriptionKey = subscriptionKeys[x]
                    const subscription = this.subscriptions[subscriptionKey]

                    if (subscription.channelKey === channelKey) {
                        const dbRef = this.$firebase.database().ref(`subscriptions/${subscriptionKey}`)
                        dbRef.set(null)
                        break
                    }
                }
            }
        },
        filters: {
            dateFormat(date) {
                return moment(date).format("MMM Do YYYY")
            }
        },
        watch: {
            channelId(val) {
                this.bindChannel(val)
            },
            channel() {
                this.bindTasks(this.channelKey)
            },
        },
    }
</script>

<style scoped>
    .item {
        @apply flex flex-col text-left w-full h-full transition ease-in-out duration-200
    }

    .item:hover {
        @apply bg-gray-200 border border-gray-400
    }

    .selectable {
        @apply transition ease-in-out duration-200
    }

    .selectable:active {
        @apply transform scale-95
    }
</style>
