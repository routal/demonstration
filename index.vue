<template>
    <section
        :class="{'-preloader': loading }"
        class="mercury wrapper wrapper_no-padding-top"
    >
        <div class="mercury__update">
            <Button
                :has-animation="updateInformation.status"
                :disabled="updateInformation.status"
                image-name="refresh.svg"
                class="button_green"
                text="Обновить данные"
                :action="getUpdateData"
            />

            <template v-if="updateInformation.status">
                Идет обновление…
            </template>

            <template v-if="!updateInformation.status && dataNotLoaded && updateInformation.date.length === 0">
                Данные не были загружены
            </template>

            <div v-if="updateInformation.date.length && !isUpdatingDate">
                Данные актуальны на
                <div>{{ actualDate }}</div>
            </div>

            <div class="mercury__update-failed" v-if="updateInformation.date.length < 1 && countGetUpdate > 0">
                Данные получить не удалось
            </div>
        </div>

        <div class="mercury__filters-container filters-container-js">
            <div class="mercury__filters-wrapper filters-container-fixed-js">
                <div class="mercury__filters">
                    <Input
                        class="mercury__search"
                        height="100%"
                        :value="mercuryPayload.query"
                        placeholder="Поиск по таблице"
                        width-icon="18px"
                        @input="onSearch($event)"
                    />
                    <SelectWithCheckbox
                        class="mercury__senders"
                        v-if="senders && senders.length && (activeSenders.length ? activeSenders.length : true)"
                        active-company="Отправитель"
                        :list-active-company="activeSenders"
                        :lists="senders"
                        plural-name="отправителя"
                        @changeSelectItem="setSendersList($event)"
                    />
                    <SelectWithCheckbox
                        class="mercury__statuses"
                        v-if="statuses && statuses.length && (activeStatuses.length ? activeStatuses.length : true)"
                        active-company="Статус"
                        :list-active-company="activeStatuses"
                        :lists="statuses"
                        plural-name="статуса"
                        @changeSelectItem="setStatus($event)"
                    />
                    <date-picker
                        class="mercury__date-picker"
                        :value="mercuryPayload.dateIssue"
                        prefix-class="xmx"
                        range
                        format="DD.MM.YYYY"
                        title-format="DD.MM.YYYY"
                        value-type="YYYY-MM-DD"
                        placeholder="Дата оформления ВСД"
                        @change="setIssueDate($event)"
                    />
                </div>
                <div class="mercury__filters-buttons">
                    <Button
                        image-name="table-accept.svg"
                        class="button_green"
                        :disabled="repaySelectedActionActive"
                        text="Погасить выбранные"
                        :action="repaySelectedVSD"
                    />
                    <Button
                        image-name="table-accept.svg"
                        class="button_center-content"
                        text="Погасить непогашенные ВСД"
                        :action="repayOutstandingVSD"
                    />
                </div>
            </div>
        </div>


        <div class="header-container-js">
            <div class="mercury__header header-js">
                <Checkbox
                    :is-hide-label="true"
                    @handleCheck="setAllSelected"
                />
                <div class="text text_medium text_font-15">
                    <Sorting
                        area="sender"
                        title="Отправитель"
                        @changeSort="sortingColumn($event)"
                    />
                </div>
                <div class="text text_medium text_font-15">
                    <Sorting
                        area="productTitle"
                        title="Название товара"
                        @changeSort="sortingColumn($event)"
                    />
                </div>
                <div class="text text_medium text_font-15">
                    <Sorting
                        area="status"
                        title="Статус"
                        @changeSort="sortingColumn($event)"
                    />
                </div>
                <div class="text text_medium text_font-15">
                    <Sorting
                        area="issueDate"
                        title="Дата оформления"
                        @changeSort="sortingColumn($event)"
                    />
                </div>
                <div class="text text_medium text_font-15">
                    <Sorting
                        area="uuid"
                        title="Внешний код ВСД"
                        @changeSort="sortingColumn($event)"
                    />
                </div>
                <div class="text text_medium text_font-15">
                    <Sorting
                        area="waybillNumber"
                        title="Номер накладной"
                        @changeSort="sortingColumn($event)"
                    />
                </div>
                <div class="text text_medium text_font-15">
                    Количество товара
                </div>
            </div>
        </div>

        <div
            class="mercury__row content-js"
            v-for="document in documents"
            :key="document.uuid"
        >
            <Checkbox
                :is-hide-label="true"
                :value="document.uuid"
                :is-checked="selectedItems.includes(document.uuid)"
                @handleCheck="setSelected(document.uuid)"
            />
            <div class="text">
                {{ document.sender }}
            </div>
            <div class="text">
                {{ document.productTitle }}
            </div>
            <div class="text">
                <StatusBadge
                    :class="{
                        'status-badge_grey': document.status === 'Закрыт' || document.status === 'Погашен',
                        'status-badge_purple': document.status === 'В обработке',
                        'status-badge_green': document.status === 'Оформлен',
                        'status-badge_red': document.status === 'Аннулирован' || document.status === 'Отказано',
                    }"
                    :is-active-icon="true"
                    :title="document.status"
                    custom-class="status-badge_icon"
                />
            </div>
            <div class="text">
                {{ formatDate(document.issueDate, 'DD.MM.YYYY') }}
            </div>
            <div class="text">
                <textarea
                    :value="document.uuid"
                    class="mercury__textarea"
                >
                </textarea>
            </div>
            <div class="text">
                {{ document.waybillNumber }}
            </div>
            <div class="text">
                {{ document.productQuantity }}
            </div>
        </div>

        <Pagination
            v-if="documents.length"
            :page="pagination.page"
            :pages="pagination.pages"
            @paginate="onPaginate"
        />
    </section>
</template>
<script>
import { mapActions, mapState, mapMutations } from 'vuex';
import Input from '../../components/Input';
import '@/src/styles/wrapper.scss';
import '@/src/styles/text.scss';
import './styles/mercury.scss';
import Button from '../../components/Button';
import '@/src/styles/datepicker.scss';
import 'vue2-datepicker/locale/ru.js';
import DatePicker from 'vue2-datepicker';
import Checkbox from '../../components/Checkbox/Checkbox';
import StatusBadge from '../../components/StatusBadge';
import formatDate from '../../helpers/formatDate';
import Pagination from '@/components/Pagination';
import SelectWithCheckbox from '../../components/SelectWithCheckbox/SelectWithCheckbox';
import Sorting from '../../components/Sorting';

export default {
    name: 'Mercury',
    components: {
        Sorting,
        DatePicker,
        SelectWithCheckbox,
        Pagination,
        Input,
        Button,
        Checkbox,
        StatusBadge
    },
    data: () => ({
        activeStatuses: [],
        activeSenders: [],
        countGetUpdate: 0,
        isUpdatingDate: false,
        loading: false,
        mercuryPayload: {
            page: 1,
            limit: 30,
            query: '',
            sort: 'DESC',
            sortBy: 'issueDate',
            sender: '',
            senderFilter: '',
            statusFilter: '',
            status: '',
            dateIssue: [],
            dateIssueTo: '',
            dateIssueFrom: ''
        },
        allActive: false,
        filtrate: {
            loading: false,
            attributesLoaded: false,
            attributes: {
                sender: [],
                status: []
            },
            errors: {
                sender: '',
                status: '',
                issueDate: ''
            }
        },
        selectedItems: [],
        routerFilters: ''
    }),
    computed: {
        ...mapState('company/company', ['company']),
        ...mapState({
            documents: (state) => state['mercury'].documents,
            pagination: (state) => state['mercury'].pagination,
            statuses: (state) => state['mercury'].statuses,
            senders: (state) => state['mercury'].senders,
            updateInformation: (state) => state['mercury'].updateInformation,
            information: (state) => state['mercury'].information
        }),

        actualDate() {
            const options = {
                year: 'numeric',
                month: 'numeric',
                day: 'numeric',
                timezone: 'UTC',
                hour: 'numeric',
                minute: 'numeric'
            };

            return new Date(this.updateInformation.date).toLocaleTimeString('ru', options);
        },

        repaySelectedActionActive() {
            return this.selectedItems.length < 1;
        },

        dataNotLoaded() {
            return this.information.setting_exist && this.information.documents_exist;
        }
    },
    watch: {
        company() {
            this.clearFiltersArea();
        }
    },
    created() {
        this.setFiltersFromUrl();
    },
    mounted() {
        this.fetchDocuments();
        this.GET_SENDERS();
        this.GET_MERCURY_INFORMATION();
    },
    beforeDestroy() {
        this.setUpdateInformation({ area: 'status', value: false });
        this.setUpdateInformation({ area: 'date', value: '' });
        this.clearFiltersArea();
    },
    methods: {
        ...mapMutations('mercury', ['setUpdateInformation']),
        ...mapActions('mercury', ['GET_DOCUMENTS', 'SEND_REPAY_VSD', 'GET_SENDERS', 'GET_UPDATE_STATUS', 'GET_STATUS_DATA', 'GET_MERCURY_INFORMATION']),

        formatDate,

        setFiltersFromUrl() {
            const { search, status, sender, dateIssueTo, dateIssueFrom } = this.$route.query;
            if (sender) {
                this.activeSenders = JSON.parse(sender);
                this.mercuryPayload.senderFilter = sender;
            }

            if (status) {
                this.activeStatuses = JSON.parse(status);
                this.mercuryPayload.statusFilter = status;
            }

            this.mercuryPayload.query = search || '';
            this.mercuryPayload.dateIssueTo = dateIssueTo || '';
            this.mercuryPayload.dateIssueFrom = dateIssueFrom || '';

            if (dateIssueTo && dateIssueFrom) {
                this.mercuryPayload.dateIssue = [dateIssueFrom, dateIssueTo];
            }
        },

        clearFiltersArea() {
            this.mercuryPayload.dateIssue = [];
            this.mercuryPayload.query = '';
        },

        setIssueDate(dates) {
            const [dateIssueFrom, dateIssueTo] = dates;

            this.mercuryPayload.dateIssue = dates;

            if (dateIssueFrom === null && dateIssueTo === null) {
                this.mercuryPayload.dateIssueTo = '';
                this.mercuryPayload.dateIssueFrom = '';
            } else {
                this.mercuryPayload.dateIssueTo = dateIssueTo;
                this.mercuryPayload.dateIssueFrom = dateIssueFrom;
            }

            this.setRouterFilters();
            this.fetchDocuments();
        },

        setSendersList(senders) {
            let senderString = '';
            let senderFilter = [];

            senders.forEach((item) => {
                senderString += `&senderList[]=${ item.title }`;
                senderFilter.push(item.title);
            });

            this.mercuryPayload.sender = senderString;
            this.mercuryPayload.senderFilter = JSON.stringify(senderFilter);
            this.setRouterFilters();
            this.fetchDocuments();
        },

        setStatus(statuses) {
            let status = '';
            let statusFilter = [];

            statuses.forEach((item) => {
                status += `&statusList[]=${ item.key }`;
                statusFilter.push(item.title);
            });

            this.mercuryPayload.status = status;
            this.mercuryPayload.statusFilter = JSON.stringify(statusFilter);
            this.setRouterFilters();
            this.fetchDocuments();
        },

        setRouterFilters() {
            this.$router.push({
                query: {
                    search: this.mercuryPayload.query,
                    status: this.mercuryPayload.statusFilter,
                    sender: this.mercuryPayload.senderFilter,
                    dateIssueTo: this.mercuryPayload.dateIssueTo,
                    dateIssueFrom: this.mercuryPayload.dateIssueFrom
                }
            });
        },

        setSelected(uuid) {
            if (this.selectedItems.includes(uuid)) {
                this.selectedItems = this.selectedItems.filter(item => item !== uuid);
            } else {
                this.selectedItems.push(uuid);
            }
        },

        setAllSelected() {
            this.allActive = !this.allActive;

            if (this.allActive) {
                this.selectedItems = this.documents.map((document) => document.uuid);

                return;
            }

            this.selectedItems = [];
        },

        sortingColumn(field) {
            if (this.mercuryPayload.sortBy === field) {
                this.mercuryPayload.sort = this.mercuryPayload.sort === 'DESC' ? 'ASC' : 'DESC';
            } else {
                this.mercuryPayload.sortBy = field;
                this.mercuryPayload.sort = 'DESC';
            }

            this.fetchDocuments();
        },

        fetchDocuments() {
            this.loading = true;
            this.GET_DOCUMENTS(this.mercuryPayload)
                .then(() => {
                    this.loading = false;
                });
        },

        repaySelectedVSD() {
            const documentIds = {
                documentIds: this.selectedItems
            };

            this.SEND_REPAY_VSD(documentIds)
                .then(() => {
                    this.selectedItems = [];
                    this.fetchDocuments();
                });
        },
        repayOutstandingVSD() {
            const unredeemed = {
                unredeemed: true
            };

            this.SEND_REPAY_VSD(unredeemed)
                .then(() => {
                    this.selectedItems = [];
                    this.fetchDocuments();
                });
        },
        onPaginate(page) {
            this.mercuryPayload.page = page;
            this.selectedItems = [];
            this.fetchDocuments();
            document.getElementById('main-scroll')
                .scroll(0, 0);
        },
        onSearch(event) {
            this.mercuryPayload.page = 1;
            this.mercuryPayload.sortBy = '';
            this.mercuryPayload.sort = 'DESC';
            this.mercuryPayload.query = event.target.value;
            this.setRouterFilters();
            this.fetchDocuments();
        },
        getUpdateData() {
            this.countGetUpdate += 1;
            this.setUpdateInformation({ area: 'status', value: true });
            this.GET_UPDATE_STATUS()
                .then(() => {
                    const intervalUpdateData = setInterval(() => {
                        this.GET_STATUS_DATA()
                            .then(() => {
                                if (!this.updateInformation.status) {
                                    clearInterval(intervalUpdateData);
                                }
                            });
                    }, 5000);
                });
        }
    },
    breadcrumbs() {
        return [
            {
                title: 'Меркурий'
            }
        ];
    }
};
</script>
