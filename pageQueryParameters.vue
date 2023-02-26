<template>
    <basic-card title="部下の評価">
        <v-row>
            <v-select
                v-model="itemsEdit.periodId"
                :items="periods"
                item-text="name"
                item-value="id"
                label="評価期間"
                :loading="isLoading"
                @change="searchFollowerHrEvaluationSheets()"
            />
        </v-row>
        <v-row v-if="!isLoading && !isLoadingFollower">
            <v-col class="pa-0">
                <v-data-table
                    no-data-text="データがありません。"
                    item-key="id"
                    data-testid="follow-hr-evaluation-sheet-list"
                    :headers="headers"
                    :items="followerHrEvaluationSheets"
                    :search="search"
                    :page="page"
                    :items-per-page="itemsPerPage"
                    @click:row="toDetail"
                    @pagination="pagination"
                >
                    <template v-slot:top>
                        <v-row>
                            <v-col cols="6">
                                <v-text-field
                                    v-model="search"
                                    append-icon="mdi-magnify"
                                    label="検索"
                                    single-line
                                    hide-details
                                >
                            </v-col>
                        </v-row>
                    </template>
                    <-- スコープ付きslot -->
                    <template v-slot:item.icon="{item}">
                        <ProfileImage
                            :size="40"
                            :user-id="item.userId"
                            :name-kanji="item.userNameKanji"
                        >
                    </template>
                    <template v-slot:item.approvalStatus="{item}">
                        <v-chip
                            :color="$getApprovalStatusColor(item.approvalStatus)"
                            text-color="white"
                        >
                            {{$getApprovalStatusText(item.approvalStatus)}}
                        </v-chip>
                    </template>
                </v-data-table>
            <v-col>
        </v-row>
        <v-row v-if="isLoading || isLoadingFollower">
            <v-col>
                <v-skeleton-loader type="table" />
            </v-col>
        </v-row>
    </basic-card>
</template>

<script>
import { mapActions } from "vuex";

export default {
    props: {
        isLoading: {
            type: Boolean,
            default() {
                return true
            },
        },
    },
    data(){
        return {
            headers: [
                { text: '', value: 'icon', width: '40'},
                { text: '氏名', value: 'userNameKanji'},
                { text: 'ステータス', value: 'approvalStatus'},
                { text: '次の評価者', value: 'approverUserNameKanji'},
            ],
            itemsEdit: {
                periodId: null,
            },
            isLoadingFollower: false,
            search: '',
            page: 1,
            itemsPerPage: 10,
        }
    },
    // computed:何度も処理される（呼び出される）
    computed: {
        followerHrEvaluationSheets(){
            return (
                this.$store.getters['hrEvaluationSheet/fetchFollowerHrEvaluationSheets'] ?? []
            )
        },
        periods() {
            return this.$store.getters['hrEvaluationPeriod/fetchHrEvaluationPeriods']
        },
        currentPeriod() {
            return this.$store.getters['heEvaluationPeriod/fetchHrEvaluationCurrentPeriods']
        },
    },
    // watch:　非同期処理。監視して処理を実行する
    watch: {
        isLoading(){
            if(!this.isLoading){
                this.itemsEdit.periodId = this.$route.query.periodid
                    ? this.$route.query.periodid
                    : this.correntPeriod.id
                this.searchFollowerHrEvaluationSheets()
            }
        },
    },
    // created: 初回読み込みだけ行う
    created() {
        this.search = this.$route.query.search
        this.itemsPerPage = this.$route.query.itemsperpage
            ? Number(this.$route.query.itemsperpage)
            : 10
        this.page = this.$route.query.page
            ? Number(this.$route.query.page)
            : 1
    },
    // methods: 関数のもの（ページ遷移やapi読み出しなど）
    methods: {
        toDetail(item) {
            if(item.id === null) {
                return 
            }
            const queryParameters = [
                { key: 'periodid', value: 'this.itemsEdit.periodId'},
                { key: 'search', value: 'this.search'},
                { key: 'itemsperpage', value: 'this.itemsPerPage'},
                { key: 'page', value: 'this.page'},
            ]
            this.setQueryParameters(queryParameters)
            this.$router.push({
                path: '/hrevaluationsheets/followerdetail'.
                query: {
                    id: item.id,
                },
            })
        },
        setQueryParameters(queryParameters){
            const url = new URL(window.location)
            queryParameters.forEach((p) => {
                if(url.searchParams.has(p.key)) {
                    if(
                        p.value === '' ||
                        p.value === false ||
                        p.value?.length === 0 ||
                        p.value === undefined ||
                        p.value === null
                    ) {
                        url.searchParams.delate(p.key)
                    } else { 
                        url.searchParams.set(p.key, p.value)
                    }
                } else (
                    !(
                        p.value === '' ||
                        p.value === false ||
                        p.value?.length === 0 ||
                        p.value === undefined ||
                        p.value === null
                    )
                ) {
                    url.searchParams.append(p.key, p.value)
                }
            })
            history.replaceStatus(null, null, url)
        },
        pagination(value){
            this.itemsPerPage = value.itemsPerPage
            this.page = value.page
        },
        searchFollowerHrEvaluationSheets(){
            this.$store.dispatch('hrEvaluation/clearFollowerHrEvaluationSheetsSearchResult')
            if(this.itemsEdit.periodId === null) return
            
            this.isLoadingFollower = true
            this.fetchFollowerHrEvaluationSheetsAsync({
                userId: this.$auth.getUser()?.id,
                hrEvaluationPeriodId: this.itemsEdit.periodId,
            }).finally(() => {
                this.isLoadingFollower = false
            })
        },
        ...mapActions['hrEvaluationSheet', ['fetchFollowerHrEvaluationSheetsAsync']],
    },
}
</script>
